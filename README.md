# Training a LLM to Understand SSSD's Log Files

Working with large and very large log files is usually problematic. This proof of concept tries to train a Large Language Model (LLM) to understand the log files produced by SSSD and help people to work with them.

The initial idea was to fine-tune a Small Language Model (SLM), but it ended-up being much more simple to access a LLM and use Retrieval Augmented Generation (RAG) on it. Once the training material is complete, we may try to fine-tune a SLM.

In this first try, only KCM logs are recognized. The rest still remains to be done.

We worked with Gemini and provided the training file. Separately we provide the log file and asked questions about it. As Gemini does not accept `.log` files, we were forced to rename them to `.txt`.

This is a transcript of a chat session.

---
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

---

## Future Work
The initial training material is rudimentary and requires significant enhancement to be effective. This marks merely the foundational phase of its development.

Here's what still needs to be done.

  * **Complete the KCM information**  
Add more details about specific operations that were not described.  
Improve the description of instances. Currently only whole, non-rated files are supported.  
Describe a start-up phase without a previous shutdown phase as a crash.


  * **Training for other components**  
The other components weren’t even mentioned.  
They need to be described as it was done for KCM.


  * **Error detection**  
Errors are not currently detected.  
Configuration errors logged at start-up must be detected.  
Runtime errors must also be detected.
