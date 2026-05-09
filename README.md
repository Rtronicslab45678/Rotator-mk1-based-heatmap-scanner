# Rotator-mk1-based-heatmap-scanner
simple python program to control and oprate sarcnet rotator mk1 to scan a snake scan patern and also gives UDP sync triggers to a port and host for external softweres to record triggers and for the porpus of generating RF scanned heat mapping 


coppy from here ==>




import serial
import time
import sys
import socket
import msvcrt

# ==========================================================
# RF HEATMAP CAMERA MK1
# ROBUST ROTATOR CONTROL ENGINE
# ==========================================================
#
# VERIFIED WORKING WITH:
# - Rotator MK1
# - Sarknet firmware
# - Easycom2-like protocol
# - lowercase "m" telemetry command
#
# ==========================================================
# TELEMETRY FORMAT OBSERVED
# ==========================================================
#
# Example:
#
# 1,29,0,30,1,0,1,-1
#
# FIELD MAP:
#
# [0] current az
# [1] current el
# [2] target az
# [3] target el
# [4] windup
# [5] unused / weird firmware field
# [6] az error
# [7] el error
#
# ==========================================================
# FEATURES
# ==========================================================
#
# ✔ Verified telemetry parser
# ✔ Handles fragmented serial packets
# ✔ Robust command resend
# ✔ Target verification
# ✔ Position lock verification
# ✔ Snake raster scan
# ✔ Live telemetry display
# ✔ Q abort
# ✔ ESC abort
# ✔ Timeout protection
# ✔ Safe home return
# ✔ UDP sync triggers
# ✔ Minimal CPU usage
# ✔ No external keyboard library needed
#
# ==========================================================


# ==========================================================
# USER SETTINGS
# ==========================================================

COM_PORT = "COM6"
BAUD_RATE = 9600

# ==========================================================
# SCAN AREA
# ==========================================================

AZ_START = 30
AZ_END = 100

EL_START = 30
EL_END = 0

AZ_STEP = 1
EL_STEP = 1

# ==========================================================
# LOCK SETTINGS
# ==========================================================

ERROR_THRESHOLD = 1

STABLE_PACKETS_REQUIRED = 10

VERIFY_TIMEOUT = 8

SETTLE_TIMEOUT = 45

MAX_COMMAND_RETRIES = 5

STEP_DELAY = 3

# ==========================================================
# UDP SYNC
# ==========================================================

UDP_IP = "127.0.0.1"
UDP_PORT = 5005

# ==========================================================
# GLOBALS
# ==========================================================

abort_requested = False

serial_buffer = ""

latest_data = None

# ==========================================================
# UDP SOCKET
# ==========================================================

udp_socket = socket.socket(
    socket.AF_INET,
    socket.SOCK_DGRAM
)

# ==========================================================
# OPEN SERIAL PORT
# ==========================================================

print("\n================================================")
print(" RF HEATMAP CAMERA MK1")
print(" ROBUST ROTATOR CONTROL ENGINE")
print("================================================\n")

print("Opening serial port...\n")

try:

    ser = serial.Serial(
        port=COM_PORT,
        baudrate=BAUD_RATE,
        timeout=0.1,

        bytesize=serial.EIGHTBITS,
        parity=serial.PARITY_NONE,
        stopbits=serial.STOPBITS_ONE,

        xonxoff=False,
        rtscts=False,
        dsrdtr=False
    )

except Exception as e:

    print("FAILED TO OPEN SERIAL PORT\n")
    print(e)

    sys.exit()

# Prevent auto-reset
ser.dtr = False
ser.rts = False

# Allow firmware startup
time.sleep(3)

# Clear garbage
ser.reset_input_buffer()
ser.reset_output_buffer()

print("SERIAL PORT OPENED\n")

# ==========================================================
# START TELEMETRY STREAM
# ==========================================================

print("Starting telemetry stream...\n")

# IMPORTANT:
# firmware REQUIRES lowercase m
ser.write(b"m\r\n")

ser.flush()

time.sleep(1)

# ==========================================================
# ABORT CHECK
# ==========================================================

def check_abort():

    global abort_requested

    try:

        if msvcrt.kbhit():

            key = msvcrt.getch()

            try:
                key = key.decode().lower()
            except:
                return False

            # Q abort
            if key == "q":

                abort_requested = True

                print("\n\nQ ABORT REQUESTED\n")

                return True

            # ESC abort
            if ord(key) == 27:

                abort_requested = True

                print("\n\nESC ABORT REQUESTED\n")

                return True

    except:
        pass

    return False

# ==========================================================
# UDP SYNC TRIGGER
# ==========================================================

def send_sync(message):

    try:

        udp_socket.sendto(
            message.encode(),
            (UDP_IP, UDP_PORT)
        )

    except:
        pass

# ==========================================================
# PARSE TELEMETRY
# ==========================================================

def parse_telemetry(line):

    try:

        line = line.strip()

        if not line:
            return None

        # Ignore firmware banner text
        if "Monitoring" in line:
            return None

        parts = line.split(",")

        # VERIFIED:
        # firmware sends 8 fields
        if len(parts) < 8:
            return None

        data = {

            "current_az": float(parts[0]),
            "current_el": float(parts[1]),

            "target_az": float(parts[2]),
            "target_el": float(parts[3]),

            "windup": float(parts[4]),

            # IMPORTANT:
            # parts[5] appears junk/unused
            "err_az": float(parts[6]),
            "err_el": float(parts[7])
        }

        return data

    except:
        return None

# ==========================================================
# READ TELEMETRY
# ==========================================================

def read_telemetry():

    global serial_buffer
    global latest_data

    try:

        raw = ser.read(ser.in_waiting or 1)

        if not raw:
            return None

        text = raw.decode(errors="ignore")

        serial_buffer += text

        while "\n" in serial_buffer:

            line, serial_buffer = serial_buffer.split("\n", 1)

            line = line.strip()

            if not line:
                continue

            data = parse_telemetry(line)

            if data is None:
                continue

            latest_data = data

            return data

    except:
        return None

    return None

# ==========================================================
# WAIT FOR INITIAL TELEMETRY
# ==========================================================

print("Waiting for telemetry lock...\n")

startup_time = time.time()

while True:

    if check_abort():
        sys.exit()

    data = read_telemetry()

    if data is not None:

        print("TELEMETRY LOCK ACQUIRED\n")
        break

    if time.time() - startup_time > 15:

        print("NO VALID TELEMETRY RECEIVED\n")

        print("CHECK:")
        print("- lowercase m command")
        print("- baud rate")
        print("- COM port")
        print("- Putty closed")
        print("- firmware telemetry mode")

        ser.close()

        sys.exit()

# ==========================================================
# SEND MOVEMENT COMMAND
# ==========================================================

def send_command(az, el):

    cmd = f"{az} {el}\r\n"

    ser.write(cmd.encode())

    ser.flush()

# ==========================================================
# VERIFY TARGET ACCEPTED
# ==========================================================

def verify_target(az, el):

    start = time.time()

    stable = 0

    while True:

        if check_abort():
            return False

        data = read_telemetry()

        if data is None:
            continue

        target_az = int(data["target_az"])
        target_el = int(data["target_el"])

        # Correct target received
        if target_az == az and target_el == el:

            stable += 1

        else:

            stable = 0

        # Require multiple confirmations
        if stable >= 3:

            return True

        # Timeout
        if time.time() - start > VERIFY_TIMEOUT:

            return False

# ==========================================================
# RELIABLE MOVE FUNCTION
# ==========================================================

def move_to(az, el):

    for attempt in range(MAX_COMMAND_RETRIES):

        print(
            f"\nSENDING COMMAND -> {az} {el} "
            f"(attempt {attempt+1})"
        )

        send_command(az, el)

        # IMPORTANT:
        # give firmware parser time
        time.sleep(0.2)

        verified = verify_target(az, el)

        if verified:

            print("TARGET VERIFIED\n")

            return True

        print("TARGET VERIFY FAILED -> RESENDING")

    print("\nFAILED TO SEND VALID TARGET\n")

    return False

# ==========================================================
# WAIT FOR POSITION LOCK
# ==========================================================

def wait_for_lock():

    start = time.time()

    stable_counter = 0

    while True:

        if check_abort():
            return False

        data = read_telemetry()

        if data is None:
            continue

        current_az = data["current_az"]
        current_el = data["current_el"]

        target_az = data["target_az"]
        target_el = data["target_el"]

        err_az = data["err_az"]
        err_el = data["err_el"]

        timestamp = time.strftime("%H:%M:%S")

        print(
            f"\r[{timestamp}] "
            f"CUR({current_az:.0f},{current_el:.0f}) "
            f"TGT({target_az:.0f},{target_el:.0f}) "
            f"ERR({err_az:.0f},{err_el:.0f}) ",
            end=""
        )

        # Lock detection
        if (
            abs(err_az) <= ERROR_THRESHOLD
            and
            abs(err_el) <= ERROR_THRESHOLD
        ):

            stable_counter += 1

        else:

            stable_counter = 0

        # Require stable packets
        if stable_counter >= STABLE_PACKETS_REQUIRED:

            print("\nPOSITION LOCKED\n")

            return True

        # Timeout protection
        if time.time() - start > SETTLE_TIMEOUT:

            print("\nLOCK TIMEOUT\n")

            return False

# ==========================================================
# MAIN SCAN LOOP
# ==========================================================

print("================================================")
print(" STARTING RASTER SCAN")
print(" PRESS Q OR ESC TO ABORT")
print("================================================\n")

scan_direction = 1

point_counter = 0

scan_start_time = time.time()

try:

    # ======================================================
    # ELEVATION LOOP
    # ======================================================

    for el in range(
        EL_START,
        EL_END - 1,
        -EL_STEP
    ):

        if abort_requested:
            break

        print(f"\n========== ELEVATION {el} ==========\n")

        # ==================================================
        # SNAKE SCAN
        # ==================================================

        if scan_direction == 1:

            azimuths = range(
                AZ_START,
                AZ_END + 1,
                AZ_STEP
            )

        else:

            azimuths = range(
                AZ_END,
                AZ_START - 1,
                -AZ_STEP
            )

        # ==================================================
        # AZIMUTH LOOP
        # ==================================================

        for az in azimuths:

            if abort_requested:
                break

            # ------------------------------------------------
            # MOVE ROTATOR
            # ------------------------------------------------

            success = move_to(az, el)

            if not success:

                print(
                    "\nSKIPPING POINT DUE TO "
                    "COMMAND FAILURE\n"
                )

                continue

            # ------------------------------------------------
            # WAIT FOR POSITION LOCK
            # ------------------------------------------------

            locked = wait_for_lock()

            if not locked:

                print(
                    "\nSKIPPING POINT DUE TO "
                    "LOCK FAILURE\n"
                )

                continue

            # ------------------------------------------------
            # MEASUREMENT SYNC
            # ------------------------------------------------

            print("MEASURE_START")

            send_sync(
                f"MEASURE_START,{az},{el}"
            )

            # PLACE RTLSDR CODE HERE LATER

            time.sleep(STEP_DELAY)

            print("MEASURE_STOP")

            send_sync(
                f"MEASURE_STOP,{az},{el}"
            )

            # ------------------------------------------------
            # STATUS
            # ------------------------------------------------

            point_counter += 1

            elapsed = time.time() - scan_start_time

            print(
                f"POINT #{point_counter} COMPLETE | "
                f"AZ={az} EL={el} | "
                f"SCAN TIME={elapsed:.1f}s"
            )

        # Reverse snake direction
        scan_direction *= -1

    print("\n================================================")
    print(" SCAN COMPLETE")
    print("================================================\n")

except KeyboardInterrupt:

    print("\nCTRL+C INTERRUPT\n")

except Exception as e:

    print("\nMAIN LOOP ERROR\n")
    print(e)

# ==========================================================
# SAFE SHUTDOWN
# ==========================================================

finally:

    try:

        print("\nRETURNING HOME -> 0 0\n")

        move_to(0, 0)

        wait_for_lock()

    except:
        pass

    print("\nCLOSING SERIAL PORT\n")

    try:
        ser.close()
    except:
        pass

    print("PROGRAM FINISHED\n")
