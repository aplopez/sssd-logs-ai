# Training a LLM to Understand SSSD's Log Files

Working with large and very large log files is usually problematic. This proof of concept tries to train a Large Language Model (LLM) to understand the log files produced by SSSD and help people to work with them.

The initial idea was to fine-tune a Small Language Model (SLM), but it ended-up being much more simple to access a LLM and use Retrieval Augmented Generation (RAG) on it. Once the training material is complete, we may try to fine-tune a SLM.

In this first try, only KCM logs are recognized. The rest still remains to be done.

We worked with Gemini and provided the training file. Separately we provide the log file and asked questions about it. As Gemini does not accept `.log` files, we were forced to rename them to `.txt`.


## Future Work
The initial training material is rudimentary and requires significant enhancement to be effective. This marks merely the foundational phase of its development.

Here's what still needs to be done.

  * **Complete the KCM information**  
Add more details about specific operations and tasks that were not described.  


  * **Training for other components**  
The other components weren’t even mentioned.  
They need to be described as it was done for KCM.


  * **Error detection**  
Errors are not currently detected.  
Configuration errors logged at start-up must be detected.  
Runtime errors must also be detected.This is a transcript of a chat session.

## Example Chat Transcripts

### Example 1

(Provide the “Log Training Data.md” file)  
**Q: Consider this file.**  
A: I have read the "Log Training Data.md" file. How can I help you with it?

(Provide the sssd_kcm.log file renamed to sssd_kcm.txt)  
**Q: Given this KCM log file. How many KCM instances can be detected?**  
A: Two KCM instances can be detected in the provided log file.

**Q: At what time did the CID#1 started in the second instance?**  
A: The client session for CID#1 in the second instance started at 14:16:56:185268.

**Q: Which client sessions were established, by which client application, and which operations they did?**  
A: Here are the client sessions established, the client applications that established them, and the operations performed:

#### First KCM Instance:

  * CID#1  
Client Application: kinit  
Operations:  
GET_DEFAULT_CACHE  
GET_PRINCIPAL 

  * CID#2  
Client Application: kinit  
Operations:  
GET_CACHE_UUID_LIST 

  * CID#3
Client Application: klist  
Operations:  
GET_DEFAULT_CACHE  
GET_PRINCIPAL 


#### Second KCM Instance:
  * CID#1  
Client Application: klist  
Operations:  
GET_DEFAULT_CACHE  
GET_PRINCIPAL 

  * CID#2  
Client Application: kinit  
Operations:  
GET_DEFAULT_CACHE  
GET_PRINCIPAL 

  * CID#3  
Client Application: kinit  
Operations:  
GET_CACHE_UUID_LIST 

**Q: Which options were set for each instance?**  
A: Here are the options that were set for each KCM instance:
####First KCM Instance:
  * `tgt_renewal` was set to `true`.
  * `tgt_renewal_inherit` was set to `none`.
  * `krb5_renew_interval` was set to `1m`.
  * `krb5_lifetime` was set to `none`.
  * `krb5_renewable_lifetime` was set to `none`.
  * `krb5_validate` was set to `false`.
  * `krb5_canonicalize` was set to `false`.
  * `krb5_auth_timeout` was set to `0`.

#### Second KCM Instance:
  * `tgt_renewal` was set to `false`.
  * `tgt_renewal_inherit` was not explicitly logged as set, but "TGT Renewals support disabled"  implies its effective state.
  * `responder_idle_timeout` was set to a value lower than the minimum allowed (60s), so the minimum allowed value was used.

**Q: Which capabilities were used in each instance?**  
A: In both the first and second KCM instances, the capabilities used were "(nothing)".

### Example 2

(Provide the “Log Training Data.md” file)  
**Q: learn from this file.**  
A: The user has provided a document explaining how to read SSSD logs, focusing on the KCM component. This document details the generic log format, then provides specific information about KCM logs, including startup and shutdown phases, how client sessions are identified (using CID or pointer-fd pairs), and how operations are logged and identified. It also describes the "Automatic TGT Renewal" feature. I will use this information to answer future questions about SSSD logs.

(Provide a sssd_kcm.txt file with incomplete instances and a crash)  
**Q: How many instances do you find in this file?**  
A: I found 3 instances in the file.

**Q: Describe them**  
A: Here are the details of the 3 KCM instances found in the log file:

**Instance 1:**

* **Start-up:** This instance does not show a complete start-up phase as the log begins mid-operation. The earliest entry for this instance is from 2025-06-16 15:32:10:100000.
* **Operations:** It handles client operations including `GET_DEFAULT_CACHE` and `GET_PRINCIPAL` for CID#9.
* **Client Sessions:** There is one client session identified by CID#9, with pointer `0x55cffccf73d0` and file descriptor `15`. The client is `/usr/libexec/goa-identity-service` with UID 1000.
* **End:** This instance ends abruptly without a shutdown phase, suggesting a crash, as the next log entry is a new KCM instance starting.

**Instance 2:**

* **Start-up:** The start-up phase begins at `2025-06-17 14:13:10:563116` with the message "Starting with debug level = 0x2f7f0" and ends with "KCM Initialization complete" at `2025-06-17 14:13:10:564166`.
* **Configuration:** During startup, `tgt_renewal` is set to `true`, `tgt_renewal_inherit` to `none`, `krb5_renew_interval` to `1m`, `krb5_lifetime` and `krb5_renewable_lifetime` to `none`, `krb5_validate` and `krb5_canonicalize` to `false`, and `krb5_auth_timeout` to `0`.
* **Operations:** This instance handles client operations including `GET_DEFAULT_CACHE` and `GET_PRINCIPAL` for CID#1 (kinit) [cite: 13, 14, 15, 16, 17, 18, 19] and `GET_CACHE_UUID_LIST` for CID#2 (kinit)[cite: 22, 23, 24, 25]. It also processes a `klist` command for CID#3.
* **Client Sessions:**
    * CID#1, pointer `0x56006836cbe0`, fd `14`, client `kinit`, UID `0`.
    * CID#2, pointer `0x56006836d570`, fd `15`, client `kinit`, UID `0`.
    * CID#3, pointer `0x56006836cbe0`, fd `14`, client `klist`, UID `0`.
* **End:** This instance ends abruptly, as the next log entry is a new KCM instance starting.

**Instance 3:**

* **Start-up:** The start-up phase begins at `2025-06-17 14:16:56:184287` with the message "Starting with debug level = 0x2f7f0" and ends with "KCM Initialization complete" at `2025-06-17 14:16:56:184943`.
* **Configuration:** During startup, `tgt_renewal` is set to `false`, `tgt_renewal_inherit` is not explicitly set but `TGT Renewals support disabled` is logged, `responder_idle_timeout` is set to a minimum of `60s` as the configured value was too low.
* **Operations:** This instance handles client operations including `GET_DEFAULT_CACHE` and `GET_PRINCIPAL` for CID#1 (klist) and CID#2 (kinit). It also handles a `GET_CACHE_UUID_LIST` operation for CID#3 (kinit).
* **Client Sessions:**
    * CID#1, pointer `0x55fdb9fb4db0`, fd `14`, client `klist`, UID `0`.
    * CID#2, pointer `0x55fdb9fb4db0`, fd `14`, client `kinit`, UID `0`.
    * CID#3, pointer `0x55fdb9fd2800`, fd `15`, client `kinit`, UID `0`.
* **End:** This instance is still active at the end of the provided log file, and there is no shutdown phase.

