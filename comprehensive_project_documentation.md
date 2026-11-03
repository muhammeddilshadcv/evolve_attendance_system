# 🪪 ESP32 Cloud-Connected Smart Attendance System & Admin Terminal

An enterprise-ready, edge-cloud automated attendance terminal powered by an **ESP32 (38-Pin DevKit)**, **Supabase PostgreSQL backend**, **MFRC522 RFID reader**, **DS3231 RTC**, **4x3 Matrix Keypad**, **16x2 I2C LCD**, and **Telegram Bot alerts**. 

The system features real-time parent notifications, offline flash caching with auto-sync, an admin web console with live attendance feeds, role classification (Students vs. Staff), and an on-terminal physical hardware management menu.

---

## 📑 Table of Contents
1. [System Architecture](#-system-architecture)
2. [Key Features](#-key-features)
3. [Hardware Bill of Materials (BOM)](#-hardware-bill-of-materials-bom)
4. [Master Pinout & Wiring Diagram](#-master-pinout--wiring-diagram)
5. [Hardware Finite State Machine (FSM)](#-hardware-finite-state-machine-fsm)
6. [Offline Caching & Failover Engine](#-offline-caching--failover-engine)
7. [Supabase Schema, Security & Edge Functions](#-supabase-schema-security--edge-functions)
8. [Automated Telegram Parent Notification Engine](#-automated-telegram-parent-notification-engine)
9. [Admin Web Console](#-admin-web-console)
10. [Firmware Configuration & Flashing Guide](#-firmware-configuration--flashing-guide)
11. [Carrier Board PCB Specifications](#-carrier-board-pcb-specifications)
12. [License](#-license)

---

## 🏗 System Architecture

```text
  +------------------+         +-------------------+         +---------------------+
  |   RFID Card /    |         | 4x3 Matrix Keypad |         | DS3231 Real-Time    |
  |  Keyfob (13.56M) |         |  (Admin & Manual) |         | Clock (Battery-Back)|
  +--------+---------+         +---------+---------+         +----------+----------+
           |                             |                              |
           +--------------------+        |        +---------------------+
                                |        |        |
                                v        v        v
                        +--------------------------------+
                        |      ESP32 Microcontroller     |
                        |      (38-Pin NodeMCU Core)     |
                        +---------------+----------------+
                                        |
                 +----------------------+-----------------------+
                 |                                              |
                 v                                              v
      +--------------------+                         +--------------------+
      | 16x2 I2C Character |                         | Active Buzzer      |
      | LiquidCrystal LCD  |                         | Audible Feedback   |
      +--------------------+                         +--------------------+
                 |
                 | (HTTPS / REST API / TLS 1.3)
                 v
   +------------------------------------------------------------------------+
   |                        Supabase Cloud Platform                        |
   |                                                                        |
   |  +------------------------+                  +----------------------+  |
   |  | PostgreSQL Database    |                  | pg_net Async HTTP    |  |
   |  | - users                |                  | Webhooks Engine      |  |
   |  | - attendance_logs      |                  +----------+-----------+  |
   |  | - system_config        |                             |              |
   |  +-----------+------------+                             |              |
   |              |                                          |              |
   |              | (Realtime WebSockets)                    |              |
   +--------------|------------------------------------------|--------------+
                  |                                          |
                  v                                          v
      +-----------------------+                  +-----------------------+
      |  Admin Web Dashboard  |                  | Telegram Notification |
      | (Single-Page HTML5/JS)|                  | Parents / Guardians   |
      +-----------------------+                  +-----------------------+
```

---

## ⚡ Key Features

- **Sub-Second NFC Attendance**: High-speed identification via 13.56 MHz MFRC522 SPI interface.
- **Fail-Safe Offline Storage (LittleFS)**: If Wi-Fi disconnects, punches are cached to the ESP32 internal flash memory (`/offline_queue.csv`) with hardware RTC timestamps and automatically synced when Wi-Fi is restored.
- **Precision Time Synchronization**: Dual-tier time management combining NTP server synchronization over Wi-Fi with an onboard DS3231 battery-backed I2C RTC.
- **Configurable Cutoff Times**: Automatic logic to distinguish between `ON_TIME` and `LATE` arrivals (e.g., Staff cutoff at 09:50 AM, Students cutoff at 10:10 AM).
- **Automated Telegram Alerts**: Integrated PostgreSQL trigger (`pg_net`) sends automated Markdown attendance and late alerts directly to parents' smartphones.
- **Physical Terminal Admin Access**: Double-press `*` on the 4x3 keypad to trigger an on-terminal admin mode for card registration and manual attendance overrides.
- **Zero-Dependency Web Dashboard**: Native HTML5, modern CSS, and vanilla JavaScript dashboard using Supabase WebSockets for live feed updates.

---

## 📦 Hardware Bill of Materials (BOM)

| Item | Component | Specification | Quantity | LCSC Part # |
| :--- | :--- | :--- | :--- | :--- |
| **U1** | ESP32 Dev Module | 38-Pin NodeMCU (ESP-WROOM-32) | 1 | C2934560 |
| **J1** | RC522 RFID Reader | 13.56 MHz SPI Module + Keyfobs/Cards | 1 | C225508 |
| **J2** | DS3231 RTC Module | High-precision Real-Time Clock with CR2032 | 1 | C225504 |
| **J3** | 16x2 Character LCD | HD44780 with PCF8574 I2C Backpack (0x27) | 1 | C225504 |
| **J4** | 4x3 Matrix Keypad | Membrane Ribbon Connector (7-Pin, 2.54mm) | 1 | C225507 |
| **BZ1**| Active Buzzer | 5V 12mm x 9.5mm Through-Hole | 1 | C965709 |
| **J5** | Power Terminal | KF301-2P 5.08mm Screw Terminal | 1 | C8395 |
| **C1** | Electrolytic Capacitor| 100µF 16V Low ESR Radial (Filter) | 1 | C329977 |
| **C2** | Ceramic Capacitor | 100nF (0.1µF) 50V X7R (Bypass) | 1 | C14663 |

---

## 🔌 Master Pinout & Wiring Diagram

### Pin Connection Table

| Peripheral Module | Pin Name | ESP32 (38-Pin) Pin | Description / Electrical Constraints |
| :--- | :--- | :--- | :--- |
| **Power Input** | `VIN / 5V` | **Pin 19 (5V / VIN)** | Regulated 5V DC Supply Input |
| | `GND` | **Pins 14, 20, 26 (GND)**| Common System Ground |
| **RC522 RFID** | `3.3V` | **Pin 1 (3V3)** | **Must connect to 3.3V! 5V will destroy RC522** |
| | `GND` | **GND** | System Ground |
| | `RST` | **Pin 32 (GPIO 4)** | Hardware Reset Line |
| | `SDA (SS)`| **Pin 29 (GPIO 5)** | SPI Slave Select / Chip Select |
| | `SCK` | **Pin 28 (GPIO 18)**| VSPI Clock |
| | `MISO` | **Pin 27 (GPIO 19)**| VSPI Master In / Slave Out |
| | `MOSI` | **Pin 21 (GPIO 23)**| VSPI Master Out / Slave In |
| | `IRQ` | *NC* | Leave Unconnected |
| **16x2 I2C LCD** | `VCC` | **Pin 19 (5V / VIN)** | 5V Backlight and Controller Power |
| *(Address: 0x27)* | `GND` | **GND** | System Ground |
| | `SDA` | **Pin 25 (GPIO 21)**| Shared I2C Data Bus (with RTC) |
| | `SCL` | **Pin 22 (GPIO 22)**| Shared I2C Clock Bus (with RTC) |
| **DS3231 RTC** | `VCC` | **Pin 1 (3V3)** | 3.3V Power Supply |
| *(Address: 0x68)* | `GND` | **GND** | System Ground |
| | `SDA` | **Pin 25 (GPIO 21)**| Shared I2C Data Bus (with LCD) |
| | `SCL` | **Pin 22 (GPIO 22)**| Shared I2C Clock Bus (with LCD) |
| **Active Buzzer** | `Positive (+)`| **Pin 7 (GPIO 32)** | Logic HIGH = Beep Output |
| | `Negative (-)`| **GND** | System Ground |
| **4x3 Keypad** | `Row 1 (R1)` | **Pin 15 (GPIO 13)**| Matrix Row 1 (Keys: 1, 2, 3) |
| | `Row 2 (R2)` | **Pin 13 (GPIO 12)**| Matrix Row 2 (Keys: 4, 5, 6) |
| | `Row 3 (R3)` | **Pin 12 (GPIO 14)**| Matrix Row 3 (Keys: 7, 8, 9) |
| | `Row 4 (R4)` | **Pin 11 (GPIO 27)**| Matrix Row 4 (Keys: *, 0, #) |
| | `Col 1 (C1)` | **Pin 10 (GPIO 26)**| Matrix Col 1 (Keys: 1, 4, 7, *) |
| | `Col 2 (C2)` | **Pin 9 (GPIO 25)** | Matrix Col 2 (Keys: 2, 5, 8, 0) |
| | `Col 3 (C3)` | **Pin 8 (GPIO 33)** | Matrix Col 3 (Keys: 3, 6, 9, #) |

---

### Detailed ASCII Wiring Schematic

```text
                               +-----------------------------+
                               |     ESP32 DEVKIT (38-PIN)   |
                               |                             |
      +------------------------| 3V3                     VIN |<--- +5V DC Rail (External Power)
      |                        | EN                      GND |<--- Common Ground Rail (GND)
      |                        | VP (GPIO36)          GPIO23 |----> RC522 MOSI
      |                        | VN (GPIO39)          GPIO22 |----> Shared I2C SCL (LCD & RTC)
      |                        | GPIO34                 TXD0 |
      |                        | GPIO35                 RXD0 |
      |      Buzzer (+) <------| GPIO32               GPIO21 |----> Shared I2C SDA (LCD & RTC)
      |      Keypad C3  <------| GPIO33                  GND |<--- Common Ground
      |      Keypad C2  <------| GPIO25               GPIO19 |----> RC522 MISO
      |      Keypad C1  <------| GPIO26               GPIO18 |----> RC522 SCK
      |      Keypad R4  <------| GPIO27                GPIO5 |----> RC522 SDA (SS)
      |      Keypad R3  <------| GPIO14               GPIO17 |
      |      Keypad R2  <------| GPIO12               GPIO16 |
      |                        | GND                   GPIO4 |----> RC522 RST
      |      Keypad R1  <------| GPIO13                GPIO0 |
      |                        | GPIO9                 GPIO2 |
      |                        | GPIO10               GPIO15 |
      |                        | GPIO11                GPIO8 |
      |                        +-----------------------------+
      |
      |   RC522 RFID READER (3.3V)
      +-> [VCC]
          [RST] -----------------------------------------------> ESP32 GPIO 4
          [GND] -----------------------------------------------> System GND
          [MISO] ----------------------------------------------> ESP32 GPIO 19
          [MOSI] ----------------------------------------------> ESP32 GPIO 23
          [SCK] -----------------------------------------------> ESP32 GPIO 18
          [SDA/SS] --------------------------------------------> ESP32 GPIO 5

          I2C 16x2 LCD (5V) & DS3231 RTC (3.3V)
          [LCD VCC] -------------------------------------------> +5V DC Rail
          [RTC VCC] -------------------------------------------> +3.3V DC Rail
          [GND] -----------------------------------------------> System GND
          [SDA] -----------------------------------------------> ESP32 GPIO 21
          [SCL] -----------------------------------------------> ESP32 GPIO 22

          ACTIVE BUZZER (5V)
          [Pin +] ---------------------------------------------> ESP32 GPIO 32
          [Pin -] ---------------------------------------------> System GND

          4x3 MATRIX KEYPAD
          [Pin 1: R1] -----------------------------------------> ESP32 GPIO 13
          [Pin 2: R2] -----------------------------------------> ESP32 GPIO 12
          [Pin 3: R3] -----------------------------------------> ESP32 GPIO 14
          [Pin 4: R4] -----------------------------------------> ESP32 GPIO 27
          [Pin 5: C1] -----------------------------------------> ESP32 GPIO 26
          [Pin 6: C2] -----------------------------------------> ESP32 GPIO 25
          [Pin 7: C3] -----------------------------------------> ESP32 GPIO 33
```

---

## 🔄 Hardware Finite State Machine (FSM)

```text
       +--------------------------------------------------------------+
       |                                                              | (15s Inactivity Timeout
       v                                                              |  or '*' Cancel Key)
+-----------------------+   Double-Tap '*'    +-----------------------+-------+
|                       |  (within 2.0s)      |                               |
|  MODE_IDLE_ATTENDANCE +-------------------->|      MODE_ADMIN_ENTER_PIN     |
|                       |                     |                               |
+-----------+-----------+                     +---------------+---------------+
            |                                                 |
  Tap Card  |                                 Submit PIN '#'  | (RPC: verify_admin_pin)
            v                                                 v
+-----------------------+                     +-------------------------------+
| Process Attendance:   |                     |                               |
| 1. Online: Post RPC   |                     |    MODE_ADMIN_MENU_SELECT     |
| 2. Offline: LittleFS  |                     |  [1] Pair Card   [2] Override |
+-----------------------+                     +-------+---------------+-------+
                                                      |               |
                                     Press '1'        |               | Press '2'
                                                      v               v
                                      +-----------------+   +------------------+
                                      | MODE_ADMIN_PAIR |   | MODE_ADMIN_MANUAL|
                                      |    _ENTER_ID    |   |   _ATTENDANCE    |
                                      +--------+--------+   +--------+---------+
                                               |                     |
                                  Enter ID + '#'|         Enter ID +'#'|
                                               v                     v
                                      +-----------------+   +------------------+
                                      | MODE_ADMIN_PAIR |   | Mark Attendance  |
                                      |   _WAIT_CARD    |   | (ADMIN_OVERRIDE) |
                                      +--------+--------+   +------------------+
                                               |
                                     Tap Card  | (RPC: pair_rfid_card)
                                               v
                                      +-----------------+
                                      | UID Linked      |
                                      | Return to Idle  |
                                      +-----------------+
```

---

## 💾 Offline Caching & Failover Engine

If Wi-Fi drops or your router restarts:
1. `WiFi.status() != WL_CONNECTED` triggers the failover path.
2. The terminal reads the current timestamp directly from the **DS3231 RTC**.
3. It appends the entry to flash memory via LittleFS (`/offline_queue.csv`):
   ```csv
   83A412F0,RFID,2026-11-03T09:42:15+05:30
   1002,ADMIN_OVERRIDE,2026-11-03T09:44:00+05:30
   ```
4. The terminal beeps and confirms: `Saved Offline OK`.
5. Every 30 seconds during idle state, the ESP32 checks for Wi-Fi recovery. Once reconnected, it streams cached punches to the `process_attendance` RPC using their recorded timestamps and clears the file.

---

## 🗄 Supabase Schema, Security & Edge Functions

Execute this complete script in the **Supabase SQL Editor**:

```sql
-- 1. Enable pg_net for asynchronous Telegram HTTP calls
CREATE EXTENSION IF NOT EXISTS pg_net WITH SCHEMA extensions;

-- 2. Master Users Table
CREATE TABLE IF NOT EXISTS users (
    id BIGSERIAL PRIMARY KEY,
    numeric_id INT UNIQUE NOT NULL,
    rfid_uid VARCHAR(32) UNIQUE,
    name VARCHAR(100) NOT NULL,
    role VARCHAR(10) NOT NULL CHECK (role IN ('student', 'staff')),
    parent_phone VARCHAR(15),
    telegram_chat_id BIGINT,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    
    CONSTRAINT check_id_range CHECK (
        (role = 'staff' AND numeric_id BETWEEN 1 AND 999) OR
        (role = 'student' AND numeric_id >= 1000)
    )
);

CREATE INDEX IF NOT EXISTS idx_users_rfid ON users(rfid_uid);
CREATE INDEX IF NOT EXISTS idx_users_num_id ON users(numeric_id);

ALTER TABLE users ENABLE ROW LEVEL SECURITY;
CREATE POLICY "Allow read access to users" ON users FOR SELECT TO anon, authenticated USING (true);
CREATE POLICY "Allow service role full access to users" ON users FOR ALL TO service_role USING (true);

-- 3. Attendance Logs Table
CREATE TABLE IF NOT EXISTS attendance_logs (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT REFERENCES users(id) ON DELETE CASCADE,
    punch_date DATE NOT NULL,
    punch_time TIME NOT NULL,
    status VARCHAR(15) NOT NULL CHECK (status IN ('ON_TIME', 'LATE')),
    late_minutes INT DEFAULT 0,
    punch_method VARCHAR(20) NOT NULL CHECK (punch_method IN ('RFID', 'KEYPAD', 'OFFLINE_SYNC', 'ADMIN_OVERRIDE')),
    created_at TIMESTAMPTZ DEFAULT NOW(),
    
    CONSTRAINT unique_user_daily_attendance UNIQUE (user_id, punch_date)
);

CREATE INDEX IF NOT EXISTS idx_attendance_date ON attendance_logs(punch_date);

ALTER TABLE attendance_logs ENABLE ROW LEVEL SECURITY;
CREATE POLICY "Allow read access to attendance_logs" ON attendance_logs FOR SELECT TO anon, authenticated USING (true);
CREATE POLICY "Allow service role full access to attendance_logs" ON attendance_logs FOR ALL TO service_role USING (true);

-- Enable Realtime WebSockets
DO $$
BEGIN
    IF NOT EXISTS (
        SELECT 1 FROM pg_publication_tables 
        WHERE pubname = 'supabase_realtime' AND tablename = 'attendance_logs'
    ) THEN
        ALTER PUBLICATION supabase_realtime ADD TABLE attendance_logs;
    END IF;
END $$;

-- 4. System Configuration Table (Stores Dynamic Admin PIN)
CREATE TABLE IF NOT EXISTS system_config (
    key TEXT PRIMARY KEY,
    value TEXT NOT NULL,
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

INSERT INTO system_config (key, value)
VALUES ('admin_pin', '1234')
ON CONFLICT (key) DO NOTHING;

ALTER TABLE system_config ENABLE ROW LEVEL SECURITY;
CREATE POLICY "Allow read on system_config" ON system_config FOR SELECT TO anon, authenticated USING (true);

-- 5. RPC: Atomic Attendance Verification
CREATE OR REPLACE FUNCTION process_attendance(
    p_identifier TEXT,
    p_method TEXT,
    p_custom_timestamp TIMESTAMPTZ DEFAULT NULL
)
RETURNS JSONB
LANGUAGE plpgsql
SECURITY DEFINER
AS $$
DECLARE
    v_user RECORD;
    v_now_ist TIMESTAMP;
    v_current_date DATE;
    v_current_time TIME;
    v_status TEXT;
    v_late_mins INT := 0;
    v_cutoff_time TIME;
    v_gate_open TIME := '09:00:00'::TIME;
BEGIN
    IF p_custom_timestamp IS NOT NULL THEN
        v_now_ist := (p_custom_timestamp AT TIME ZONE 'Asia/Kolkata');
    ELSE
        v_now_ist := (NOW() AT TIME ZONE 'Asia/Kolkata');
    END IF;

    v_current_date := v_now_ist::DATE;
    v_current_time := v_now_ist::TIME;

    -- Enforce Gate Timing
    IF p_method <> 'ADMIN_OVERRIDE' AND v_current_time < v_gate_open THEN
        RETURN jsonb_build_object(
            'success', false,
            'code', 'GATE_CLOSED',
            'line1', 'Gate Closed',
            'line2', 'Opens at 9:00 AM'
        );
    END IF;

    -- Lookup User by RFID UID or Numeric ID
    IF p_identifier ~ '^[0-9]+$' THEN
        SELECT * INTO v_user FROM users WHERE numeric_id = p_identifier::INT AND is_active = TRUE;
    ELSE
        SELECT * INTO v_user FROM users WHERE UPPER(rfid_uid) = UPPER(p_identifier) AND is_active = TRUE;
    END IF;

    IF NOT FOUND THEN
        RETURN jsonb_build_object(
            'success', false,
            'code', 'USER_NOT_FOUND',
            'line1', 'Invalid User ID',
            'line2', 'Access Denied'
        );
    END IF;

    -- Prevent Duplicate Scans on the Same Day
    IF EXISTS (
        SELECT 1 FROM attendance_logs 
        WHERE user_id = v_user.id AND punch_date = v_current_date
    ) THEN
        RETURN jsonb_build_object(
            'success', false,
            'code', 'ALREADY_MARKED',
            'line1', SUBSTRING(v_user.name FROM 1 FOR 16),
            'line2', 'Already Marked!'
        );
    END IF;

    -- Determine Cutoff (Staff: 09:50 AM, Students: 10:10 AM)
    IF v_user.role = 'staff' THEN
        v_cutoff_time := '09:50:00'::TIME;
    ELSE
        v_cutoff_time := '10:10:00'::TIME;
    END IF;

    IF v_current_time > v_cutoff_time THEN
        v_status := 'LATE';
        v_late_mins := FLOOR(EXTRACT(EPOCH FROM (v_current_time - v_cutoff_time)) / 60);
    ELSE
        v_status := 'ON_TIME';
        v_late_mins := 0;
    END IF;

    INSERT INTO attendance_logs (
        user_id, punch_date, punch_time, status, late_minutes, punch_method
    ) VALUES (
        v_user.id, v_current_date, v_current_time, v_status, v_late_mins, p_method
    );

    RETURN jsonb_build_object(
        'success', true,
        'code', 'MARKED',
        'name', v_user.name,
        'role', v_user.role,
        'status', v_status,
        'line1', SUBSTRING(v_user.name FROM 1 FOR 16),
        'line2', CASE 
                    WHEN p_method = 'ADMIN_OVERRIDE' THEN 'Admin Marked OK'
                    WHEN v_status = 'ON_TIME' THEN 'Present ' || TO_CHAR(v_now_ist, 'hh12:MI AM')
                    ELSE 'Late (' || v_late_mins || 'm) ' || TO_CHAR(v_now_ist, 'hh12:MI')
                 END
    );
END;
$$;

-- 6. RPC: Secure PIN Verification
CREATE OR REPLACE FUNCTION verify_admin_pin(p_pin TEXT)
RETURNS JSONB
LANGUAGE plpgsql
SECURITY DEFINER
AS $$
DECLARE
    v_actual_pin TEXT;
BEGIN
    SELECT value INTO v_actual_pin FROM system_config WHERE key = 'admin_pin';
    IF v_actual_pin IS NULL THEN v_actual_pin := '1234'; END IF;

    IF p_pin = v_actual_pin THEN
        RETURN jsonb_build_object('success', true, 'msg', 'PIN Verified');
    ELSE
        RETURN jsonb_build_object('success', false, 'msg', 'Invalid PIN');
    END IF;
END;
$$;

-- 7. RPC: Secure PIN Update
CREATE OR REPLACE FUNCTION update_admin_pin(p_current_pin TEXT, p_new_pin TEXT)
RETURNS JSONB
LANGUAGE plpgsql
SECURITY DEFINER
AS $$
DECLARE
    v_actual_pin TEXT;
BEGIN
    IF p_new_pin !~ '^[0-9]{4,8}$' THEN
        RETURN jsonb_build_object('success', false, 'msg', 'PIN must be between 4 and 8 digits.');
    END IF;

    SELECT value INTO v_actual_pin FROM system_config WHERE key = 'admin_pin';
    IF p_current_pin <> COALESCE(v_actual_pin, '1234') THEN
        RETURN jsonb_build_object('success', false, 'msg', 'Current PIN verification failed.');
    END IF;

    INSERT INTO system_config (key, value, updated_at)
    VALUES ('admin_pin', p_new_pin, NOW())
    ON CONFLICT (key) DO UPDATE
    SET value = EXCLUDED.value, updated_at = EXCLUDED.updated_at;

    RETURN jsonb_build_object('success', true, 'msg', 'PIN updated successfully.');
END;
$$;

-- 8. RPC: Pair RFID Card
CREATE OR REPLACE FUNCTION pair_rfid_card(p_pin TEXT, p_numeric_id INT, p_rfid_uid TEXT)
RETURNS JSONB
LANGUAGE plpgsql
SECURITY DEFINER
AS $$
DECLARE
    v_actual_pin TEXT;
    v_existing_holder TEXT;
    v_target_user_name TEXT;
    v_clean_uid TEXT;
BEGIN
    SELECT value INTO v_actual_pin FROM system_config WHERE key = 'admin_pin';
    IF p_pin <> COALESCE(v_actual_pin, '1234') THEN
        RETURN jsonb_build_object('success', false, 'code', 'BAD_PIN', 'line1', 'Access Denied', 'line2', 'Incorrect PIN');
    END IF;

    v_clean_uid := UPPER(TRIM(p_rfid_uid));

    SELECT name INTO v_target_user_name FROM users WHERE numeric_id = p_numeric_id;
    IF v_target_user_name IS NULL THEN
        RETURN jsonb_build_object('success', false, 'code', 'USER_NOT_FOUND', 'line1', 'Error: Not Found', 'line2', 'ID Does Not Exist');
    END IF;

    SELECT name INTO v_existing_holder FROM users WHERE UPPER(rfid_uid) = v_clean_uid AND numeric_id <> p_numeric_id LIMIT 1;
    IF v_existing_holder IS NOT NULL THEN
        RETURN jsonb_build_object('success', false, 'code', 'CARD_IN_USE', 'line1', 'Card In Use!', 'line2', SUBSTRING(v_existing_holder FROM 1 FOR 16));
    END IF;

    UPDATE users SET rfid_uid = v_clean_uid WHERE numeric_id = p_numeric_id;

    RETURN jsonb_build_object('success', true, 'code', 'PAIRED', 'line1', 'Pair Successful!', 'line2', SUBSTRING(v_target_user_name FROM 1 FOR 16));
END;
$$;
```

---

## 📱 Automated Telegram Parent Notification Engine

When a record is inserted into `attendance_logs`, an asynchronous trigger calls the Telegram Bot API using the `pg_net` extension:

```sql
CREATE OR REPLACE FUNCTION notify_telegram_parent()
RETURNS TRIGGER
LANGUAGE plpgsql
SECURITY DEFINER
AS $$
DECLARE
    v_user RECORD;
    v_message TEXT;
    v_bot_token TEXT := '<YOUR_TELEGRAM_BOT_TOKEN>';
    v_url TEXT;
BEGIN
    SELECT * INTO v_user FROM users WHERE id = NEW.user_id;

    -- Only send notifications for students who have a linked Telegram account
    IF v_user.role = 'student' AND v_user.telegram_chat_id IS NOT NULL THEN
        IF NEW.punch_method = 'ADMIN_OVERRIDE' THEN
            v_message := '⚠️ *Notice: ID Card Forgotten*' || E'\n\n' ||
                         'Student: *' || v_user.name || '* (ID: ' || v_user.numeric_id || ')' || E'\n' ||
                         'Status: *Present (Admin Marked)*' || E'\n' ||
                         'Arrival Time: ' || TO_CHAR(NEW.punch_time, 'HH12:MI AM') || ' IST' || E'\n\n' ||
                         '📌 *Note:* Your ward forgot their physical ID card today. Attendance was marked manually by the administrator.';
        ELSIF NEW.status = 'ON_TIME' THEN
            v_message := '✅ *Attendance Notice*' || E'\n\n' ||
                         'Student: *' || v_user.name || '* (ID: ' || v_user.numeric_id || ')' || E'\n' ||
                         'Status: *Present (On-Time)*' || E'\n' ||
                         'Arrival Time: ' || TO_CHAR(NEW.punch_time, 'HH12:MI AM') || ' IST';
        ELSE
            v_message := '⚠️ *Late Arrival Alert*' || E'\n\n' ||
                         'Student: *' || v_user.name || '* (ID: ' || v_user.numeric_id || ')' || E'\n' ||
                         'Status: *Late by ' || NEW.late_minutes || ' minutes*' || E'\n' ||
                         'Arrival Time: ' || TO_CHAR(NEW.punch_time, 'HH12:MI AM') || ' IST';
        END IF;

        v_url := 'https://api.telegram.org/bot' || v_bot_token || '/sendMessage';

        PERFORM net.http_post(
            url := v_url,
            headers := jsonb_build_object('Content-Type', 'application/json'),
            body := jsonb_build_object(
                'chat_id', v_user.telegram_chat_id,
                'text', v_message,
                'parse_mode', 'Markdown'
            )
        );
    END IF;

    RETURN NEW;
END;
$$;

DROP TRIGGER IF EXISTS tr_attendance_telegram_alert ON attendance_logs;
CREATE TRIGGER tr_attendance_telegram_alert
AFTER INSERT ON attendance_logs
FOR EACH ROW
EXECUTE FUNCTION notify_telegram_parent();
```

---

## 💻 Admin Web Console

The dashboard (`admin.html`) is a standalone single-page web app built with vanilla HTML/CSS/JavaScript and the `@supabase/supabase-js` client.

### Features
1. **Secure PIN Gate**: Authenticates against the dynamic PIN stored in `system_config`.
2. **Classified Real-Time Feeds**: Displays live scans in separate side-by-side columns for students and staff.
3. **Directory Management**: Add new users (Staff IDs: 1–999, Student IDs: 1000+), unlink lost cards, and trigger manual check-in overrides.
4. **CSV Export**: Downloads a daily report with formatted 12-hour IST timestamps (`hh:mm AM/PM`) for spreadsheet software.

---

## 🛠 Firmware Configuration & Flashing Guide

### 1. Arduino IDE Setup
Add the ESP32 board URL to **File $\rightarrow$ Preferences $\rightarrow$ Additional Board Manager URLs**:
```text
https://raw.githubusercontent.com/espressif/arduino-esp32/gh-pages/package_esp32_index.json
```

### 2. Required Libraries
Install the following via the Arduino Library Manager:
- **LiquidCrystal_I2C** (by Frank de Brabander)
- **MFRC522** (by GithubCommunity)
- **RTClib** (by Adafruit)
- **Keypad** (by Mark Stanley, Alexander Brevig)
- **ArduinoJson** (v6.x by Benoît Blanchon)

### 3. Board Settings
- **Board**: `ESP32 Dev Module`
- **Flash Frequency**: `80MHz`
- **Partition Scheme**: `Default 4MB with spiffs (1.2MB App/1.5MB SPIFFS)`
- **Upload Speed**: `921600` (or `115200` if flashing fails)

---

## 📐 Carrier Board PCB Specifications

A custom motherboard layout avoids loose jumper wires and packages the terminal cleanly:

1. **Board Dimensions**: $100\text{ mm} \times 80\text{ mm}$ (fits standard two-layer pricing tiers).
2. **Power Distribution**:
   - Trace widths for `+5V` and `+3V3` should be **$0.8\text{ mm} - 1.0\text{ mm}$**.
   - Signal traces (SPI, I2C, Keypad) should be **$0.254\text{ mm} - 0.3\text{ mm}$**.
   - Use full **Ground Copper Pours (`GND`)** on both the top and bottom layers.
3. **Decoupling**: Place $100\mu\text{F}$ and $0.1\mu\text{F}$ capacitors close to the 5V screw terminal (`KF301-2P`) and the ESP32 `VIN` pin.
4. **Antenna Keepout**: Do not route copper traces or place ground planes directly underneath the **RC522 PCB trace antenna**.

---

## 📄 License
This project is open-source under the [MIT License](LICENSE).