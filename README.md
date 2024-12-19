# THM-Writeup-Splunk-Data-Manipulation
Writeup for TryHackMe Splunk: Data Manipulation - Advanced configuration with regex-based field extraction, event boundary fixes, and data masking.

By Ramyar Daneshgar 

### Splunk Data Manipulation Writeup

In this lab, I explored Splunk's powerful data analytics and parsing capabilities, focusing on field extractions, masking sensitive data, and resolving event boundary issues. This exercise allowed me to better understand how Splunk processes and indexes data from various sources, including custom scripts and log files. Below, I document my approach to solving the challenges presented in the lab.

---

### **Tasks Overview**

#### **Task 1: Setting Up Splunk and Creating a Simple App**
I started by creating a simple Splunk app named `DataApp`. The app structure was established within the `/opt/splunk/etc/apps` directory. Key directories included:
- `bin`: Used for custom scripts.
- `default`: Contained configuration files like `inputs.conf` and `props.conf`.
- `metadata`: Defined app metadata.

Using a Python script (`samplelogs.py`) placed in the `bin` directory, I generated sample logs for ingestion into Splunk. The `inputs.conf` was configured as follows:

```plaintext
[script:///opt/splunk/etc/apps/DataApp/bin/samplelogs.py]
index = main
source = test_log
sourcetype = testing
interval = 5
```

Restarting Splunk (`/opt/splunk/bin/splunk restart`) enabled the ingestion of these logs into the `main` index every 5 seconds.

---

#### **Task 2: Fixing Event Boundaries**
The challenge was to resolve issues where Splunk failed to correctly break logs into individual events. For VPN logs, I observed that events ended with either `CONNECT` or `DISCONNECT`. Using a regex pattern `(DISCONNECT|CONNECT)`, I configured the `props.conf` to enforce event boundaries:

```plaintext
[vpn_logs]
SHOULD_LINEMERGE = true
MUST_BREAK_AFTER = (DISCONNECT|CONNECT)
```

For multi-line logs (e.g., authentication logs starting with `[Authentication]`), I utilized the `BREAK_ONLY_BEFORE` stanza:

```plaintext
[auth_logs]
SHOULD_LINEMERGE = true
BREAK_ONLY_BEFORE = \[Authentication\]
```

After restarting Splunk, I verified that events were correctly segmented in real time.

---

#### **Task 3: Masking Sensitive Data**
Sensitive data, such as credit card numbers, was present in purchase logs. To ensure compliance with standards like PCI DSS, I masked these numbers using the `SEDCMD` stanza in `props.conf`. The configuration applied the following regex:

```plaintext
[purchase_logs]
SHOULD_LINEMERGE = true
MUST_BREAK_AFTER = (User)
SEDCMD-maskCC = s/-\d{4}-\d{4}-\d{4}/-XXXX-XXXX-XXXX/g
```

This effectively replaced card numbers with `XXXX-XXXX-XXXX`, ensuring no sensitive information was visible in indexed events.

---

#### **Task 4: Extracting Custom Fields**
Custom field extraction was critical for meaningful analysis. For VPN logs, I extracted fields such as `Username`, `Server`, and `Action` by creating a regex pattern and updating the `transforms.conf`, `props.conf`, and `fields.conf` files.

Example `transforms.conf`:

```plaintext
[vpn_custom_fields]
REGEX = User:\s([\w\s]+), Server:\s([\w\s]+), Action:\s(\w+)
FORMAT = Username::$1 Server::$2 Action::$3
WRITE_META = true
```

Updated `props.conf`:

```plaintext
[vpn_logs]
TRANSFORMS-vpn = vpn_custom_fields
```

With these configurations, I extracted the desired fields at index time. Restarting Splunk showed the parsed data in real time, enabling advanced querying and analysis.

---

### **Lessons Learned**
1. **Configuration Hierarchy and Best Practices**:
   - Understanding the role of files like `props.conf`, `transforms.conf`, and `inputs.conf` is crucial for configuring Splunk effectively. Using `fields.conf` for indexed fields adds precision.
   
2. **Regex**:
   - Regular expressions are foundational for extracting fields and identifying event boundaries. Tools like regex101.com simplify regex creation and testing.

3. **Data Privacy and Compliance**:
   - Masking sensitive data is a necessary step for maintaining compliance with data security standards. Leveraging Splunk's `SEDCMD` provides a straightforward solution for data anonymization.

4. **Real-Time Troubleshooting**:
   - Restarting Splunk after configuration changes and verifying results in the search head ensures incremental debugging and quicker issue resolution.
