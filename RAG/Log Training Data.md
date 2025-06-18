# SSSD Logs

## Generic Log

This document explains how to read SSSD’s logs. This format is the same for all the components, although each component has its own log file.

Log files are text files which have a sequence of entries. Each entry is usually a line, but could be more.

An entry starts with a timestamp, for instance:

`(2025-04-18 14:43:01): [kcm] [server_setup] (0x3f7c0): Starting with debug level = 0x2f7f0`

The timestamp is the first field and is surrounded by parenthesis. In this example the timestamp is 2025-04-18 14:43:01. Sometimes the timestamp is also called “time.” After the timestamp there is a separator colon.

The timestamp can, when the component is configured to do so, include the number of
microseconds. For instance:

`(2025-06-17  9:43:46:513841): [kcm] [kcm_recv] (0x4000): [CID#1] Client closed connection.`

The microseconds in this example are 513841 and the whole timestamp is 2025-06-17 9:43:46:513841.

The next field is enclosed between square brackets and is the component. In this case the component is “kcm.”

Follows the name of the function that logged the message. It is also enclosed in square brackets: server_setup.

The next field is the message’s log level. It is enclosed in parenthesis. In this example it is 0x3f7c0. It is an hexadecimal number and represents a bitmap.

Follows the message, which is all that remains after the colon. In this example the message is “Starting with debug level = 0x2f7f0”

When the following line does not start with a timestamp, then it is considered to be part of the message from the previous line. This applies to all the lines until a line that begins with a timestamp is found, which is a new entry.

In this example, there are three lines but only two entries:

```
(2025-06-02  9:08:44): [kcm] [server_loop] (0x3f7c0): Entering main loop under uid=378 (euid=378) : gid=375 (egid=375) with SECBIT_KEEP_CAPS = 0 and following capabilities:
  (nothing)
(2025-06-02 14:32:34): [kcm] [orderly_shutdown] (0x3f7c0): SIGTERM: killing children
```

The first and second lines are part of the same entry. In this case the message is:

```
Entering main loop under uid=378 (euid=378) : gid=375 (egid=375) with SECBIT_KEEP_CAPS = 0 and following capabilities:
  (nothing)
```

While the second entry is the third line, whose message is "SIGTERM: killing children."


## KCM Logs

KCM is one of SSSD’s components and has its own log. In this case, the component will always be "kcm."

### KCM Instances

#### KCM’s Start-Up Phase

KCM’s start-up phase starts with the entry with the message that begins with “Starting with debug level =”, and ends with the entry with the message “KCM Initialization complete.”

During the start-up phase, the value of some configuration options is logged. Entries with messages like `Option [XXXX] set to [YYYY]` tell that the option XXXX was set to the value YYYY.

For instance, the following entry tells that the option “krb5\_lifetime” was set to the value “none.”

`(2025-06-02 14:46:47): [kcm] [kcm_set_options] (0x0400): Option [krb5_lifetime] set to [none]`

#### KCM's Shutdown Phase

The shutdown phase starts with the message "Responder is being shut down" and ends with the confirmation message "Shutting down (status = 0)."

This is an example:

```
(2025-06-02 16:46:14): [kcm] [kcm_responder_ctx_destructor] (0x0400): Responder is being shut down
(2025-06-02 16:46:14): [kcm] [client_close_fn] (0x2000): Terminated client [0x5637d48615b0][14]
(2025-06-02 16:46:14): [kcm] [orderly_shutdown] (0x3f7c0): SIGTERM: killing children
(2025-06-02 16:46:14): [kcm] [orderly_shutdown] (0x3f7c0): Shutting down (status = 0)
```

If the last message does not include the "status = 0" part, it means there was an error during the shutdown phase and KCM was not properly shut down.

If the shutdown phase is preceded by the "Terminating idle responder" message, this means
that KCM has been idle for the time set to the responder_idle_timeout option and has
automatically shut down. When this message is not preceding the shutdown phase, the
shutdown was requested by an external entity (usually systemd).

#### Instance

A KCM instance happens between a start-up phase and a shutdown phase, both parts included.
A log file can include one or several instances.

It might happen that the start-up phase is not present in the log file, in which case the instances happens from the beginning of the file until the first shutdown phase.

It could also happens that the shutdown phase is not present in the log file, in which case
the instance starts in the last start-up phase up to the end of the log file.

I can also happen that neither the start-up phase or the shutdown phase are present, which
means that the log file covers a single instance.

<!-- Explain the cases of a crash -->

### Client Session

One or more operations are requested by a client application (also known as client).
When a client connects to KCM, a line like this one will be logged by the function accept_fd_handler:

`(2025-06-02 16:45:04): [kcm] [accept_fd_handler] (0x0400): [CID#23] Client [cmd kinit][uid 0][0x567d48fbab00][12] connected!`

This line includes in the message the CID (also known as "Client Identification Number" and "Client ID"). The CID appears at the beginning of the message enclosed in brackets and preceded by the string "CID#." In this example the CID is "23." This line also tells the actual client name. It can be seen after the keyword "Client", enclosed in brackets and preceded by the keyword "cmd"; in this case the client is the command kinit. The message also says which user launched the client. The UID or "user id" is a numeric value preceded by the keyword "uid" and both enclosed in brackets. In this example the UID is 0.

Another way to identify the client is by its pair pointer-fd (the fd is also known as file descriptor). In the example above the pointer is 0x567d48fbab00 and the fd is 12. Each one is enclosed in brackets and the file descriptor follows the pointer.

For instance, the following lines are also related to the same client because the line above and these two all three have the same pointer and fd:

```
(2025-06-02 16:45:04): [kcm] [get_client_cred] (0x4000): Client [0x567d48fbab00][11] creds: euid[0] egid[0] pid[23363] cmd_line['klist'].
(2025-06-02 16:45:04): [kcm] [setup_client_idle_timer] (0x4000): Idle timer re-set for client [0x567d48fbab00][11]
```

These two lines open a client session.

All the lines associated to a client are identified with the same CID or pointer-fd pair. Any operation not including the corresponding CID or pointer-fd pair, are not associated to the client.

After KCM restarts, the CID can be reused. Two client sessions with the same CID are
unrelated if a start-up happened in between.

This is an example of the klist command which contacted KCM creatin a new client section. The client identifier is "CID#26" and the pointer-fd pair is "[0x5637d48fbab0][15]".

```
(2025-06-02 16:46:01): [kcm] [get_client_cred] (0x4000): Client [0x5637d48fbab0][15] creds: euid[0] egid[0] pid[23363] cmd_line['klist'].
(2025-06-02 16:46:01): [kcm] [setup_client_idle_timer] (0x4000): Idle timer re-set for client [0x5637d48fbab0][15]
(2025-06-02 16:46:01): [kcm] [accept_fd_handler] (0x0400): [CID#26] Client [cmd klist][uid 0][0x5637d48fbab0][15] connected!
(2025-06-02 16:46:01): [kcm] [kcm_input_parse] (0x1000): [CID#26] Received message with length 4
(2025-06-02 16:46:01): [kcm] [kcm_get_opt] (0x2000): [CID#26] The client requested operation 20
(2025-06-02 16:46:01): [kcm] [kcm_cmd_send] (0x0400): [CID#26] KCM operation GET_DEFAULT_CACHE
(2025-06-02 16:46:01): [kcm] [kcm_cmd_send] (0x1000): [CID#26] 0 bytes on KCM input
(2025-06-02 16:46:01): [kcm] [kcm_op_queue_send] (0x0200): [CID#26] Adding request by 0 to the wait queue
(2025-06-02 16:46:01): [kcm] [kcm_op_queue_get] (0x1000): [CID#26] No existing queue for this ID
(2025-06-02 16:46:01): [kcm] [kcm_op_queue_send] (0x1000): [CID#26] Queue was empty, running the request immediately
(2025-06-02 16:46:01): [kcm] [kcm_op_get_default_ccache_send] (0x1000): [CID#26] Getting client's default ccache
(2025-06-02 16:46:01): [kcm] [ccdb_secdb_get_default_send] (0x2000): [CID#26] Getting the default ccache
(2025-06-02 16:46:01): [kcm] [local_db_dn] (0x2000): [CID#26] Local path for [persistent/0/default] is [cn=default,cn=0,cn=persistent,cn=kcm]
(2025-06-02 16:46:01): [kcm] [sss_sec_new_req] (0x1000): [CID#26] Local DB path is persistent/0/default
(2025-06-02 16:46:01): [kcm] [secdb_dfl_url_req] (0x2000): [CID#26] Created request for URL persistent/0/default
(2025-06-02 16:46:01): [kcm] [sss_sec_get] (0x0400): [CID#26] Retrieving a secret from [persistent/0/default]
(2025-06-02 16:46:01): [kcm] [sss_sec_get] (0x2000): [CID#26] Searching at [cn=default,cn=0,cn=persistent,cn=kcm] with scope=base
(2025-06-02 16:46:01): [kcm] [ccdb_secdb_get_default_send] (0x2000): [CID#26] Got the default ccache
(2025-06-02 16:46:01): [kcm] [ccdb_secdb_list_send] (0x2000): [CID#26] Listing all ccaches
(2025-06-02 16:46:01): [kcm] [local_db_dn] (0x2000): [CID#26] Local path for [persistent/0/ccache/] is [cn=ccache,cn=0,cn=persistent,cn=kcm]
(2025-06-02 16:46:01): [kcm] [sss_sec_new_req] (0x1000): [CID#26] Local DB path is persistent/0/ccache/
(2025-06-02 16:46:01): [kcm] [secdb_container_url_req] (0x2000): [CID#26] Created request for URL persistent/0/ccache/
(2025-06-02 16:46:01): [kcm] [sss_sec_list] (0x0400): [CID#26] Listing keys at [persistent/0/ccache/]
(2025-06-02 16:46:01): [kcm] [sss_sec_list] (0x2000): [CID#26] Searching at [cn=ccache,cn=0,cn=persistent,cn=kcm] with scope=subtree
(2025-06-02 16:46:01): [kcm] [sss_sec_list] (0x1000): [CID#26] No secrets found
(2025-06-02 16:46:01): [kcm] [ccdb_secdb_list_send] (0x2000): [CID#26] Found 0 ccaches
(2025-06-02 16:46:01): [kcm] [ccdb_secdb_list_send] (0x2000): [CID#26] Listing all caches done
(2025-06-02 16:46:01): [kcm] [kcm_op_get_default_ccache_reply_step] (0x2000): [CID#26] The default ccache is 0
(2025-06-02 16:46:01): [kcm] [kcm_cmd_done] (0x0400): [CID#26] KCM operation GET_DEFAULT_CACHE returned [0]: Success
(2025-06-02 16:46:01): [kcm] [kcm_send_reply] (0x2000): [CID#26] Sending a reply
(2025-06-02 16:46:01): [kcm] [kcm_output_construct] (0x1000): [CID#26] Sending a reply with 6 bytes of payload
(2025-06-02 16:46:01): [kcm] [queue_removal_cb] (0x0200): [CID#26] Removed queue for 0
(2025-06-02 16:46:01): [kcm] [kcm_send] (0x2000): [CID#26] All data sent!
(2025-06-02 16:46:01): [kcm] [kcm_input_parse] (0x1000): [CID#26] Received message with length 6
(2025-06-02 16:46:01): [kcm] [kcm_get_opt] (0x2000): [CID#26] The client requested operation 8
(2025-06-02 16:46:01): [kcm] [kcm_cmd_send] (0x0400): [CID#26] KCM operation GET_PRINCIPAL
(2025-06-02 16:46:01): [kcm] [kcm_cmd_send] (0x1000): [CID#26] 2 bytes on KCM input
(2025-06-02 16:46:01): [kcm] [kcm_op_queue_send] (0x0200): [CID#26] Adding request by 0 to the wait queue
(2025-06-02 16:46:01): [kcm] [kcm_op_queue_get] (0x1000): [CID#26] No existing queue for this ID
(2025-06-02 16:46:01): [kcm] [kcm_op_queue_send] (0x1000): [CID#26] Queue was empty, running the request immediately
(2025-06-02 16:46:01): [kcm] [kcm_op_get_principal_send] (0x1000): [CID#26] Requested principal 0
(2025-06-02 16:46:01): [kcm] [ccdb_secdb_getbyname_send] (0x2000): [CID#26] Getting ccache by name
(2025-06-02 16:46:01): [kcm] [local_db_dn] (0x2000): [CID#26] Local path for [persistent/0/ccache/] is [cn=ccache,cn=0,cn=persistent,cn=kcm]
(2025-06-02 16:46:01): [kcm] [sss_sec_new_req] (0x1000): [CID#26] Local DB path is persistent/0/ccache/
(2025-06-02 16:46:01): [kcm] [secdb_container_url_req] (0x2000): [CID#26] Created request for URL persistent/0/ccache/
(2025-06-02 16:46:01): [kcm] [sss_sec_list] (0x0400): [CID#26] Listing keys at [persistent/0/ccache/]
(2025-06-02 16:46:01): [kcm] [sss_sec_list] (0x2000): [CID#26] Searching at [cn=ccache,cn=0,cn=persistent,cn=kcm] with scope=subtree
(2025-06-02 16:46:01): [kcm] [sss_sec_list] (0x1000): [CID#26] No secrets found
(2025-06-02 16:46:01): [kcm] [key_by_name] (0x0080): [CID#26] The container was not found
(2025-06-02 16:46:01): [kcm] [kcm_ccdb_getbyname_done] (0x1000): [CID#26] No cache found by name
(2025-06-02 16:46:01): [kcm] [kcm_op_get_principal_getbyname_done] (0x0080): [CID#26] No credentials by that name
(2025-06-02 16:46:01): [kcm] [kcm_cmd_done] (0x0400): [CID#26] KCM operation GET_PRINCIPAL returned [1432158224]: No matching credentials found
(2025-06-02 16:46:01): [kcm] [kcm_send_reply] (0x2000): [CID#26] Sending a reply
(2025-06-02 16:46:01): [kcm] [kcm_output_construct] (0x1000): [CID#26] Sending a reply with 4 bytes of payload
(2025-06-02 16:46:01): [kcm] [queue_removal_cb] (0x0200): [CID#26] Removed queue for 0
(2025-06-02 16:46:01): [kcm] [kcm_send] (0x2000): [CID#26] All data sent!
(2025-06-02 16:46:01): [kcm] [kcm_recv] (0x4000): [CID#26] Client closed connection.
(2025-06-02 16:46:01): [kcm] [client_close_fn] (0x2000): [CID#26] Terminated client [0x5637d48fbab0][15]
```

When the client disconnects, the message "Client closed connection" is logged.
The message "Terminated client" indicates that the client session has ended and
that the pointer and file descriptor are no longer associated to this client.
Opposite to the CID, the pointer and fd can be reused, together or separated,
for other clients.

#### Operations

Clients request operation from KCM, and KCM sends back the result.

When an operation arrives, KCM logs the message "Received message with length 6".
6 indicates in this case the length of the message measured in bytes. The next
entry tells the numeric value of the operation the client requested:
"The client requested operation 8" and the lexical name is provided in the
following entry, with the message "KCM operation GET_PRINCIPAL". 8 is the numeric
value and GET_PRINCIPAL is the name of the operation. An operation is identified
by its name.

For instance: 
```
(2025-06-02 16:46:01): [kcm] [kcm_input_parse] (0x1000): [CID#26] Received message with length 6
(2025-06-02 16:46:01): [kcm] [kcm_get_opt] (0x2000): [CID#26] The client requested operation 8
(2025-06-02 16:46:01): [kcm] [kcm_cmd_send] (0x0400): [CID#26] KCM operation GET_PRINCIPAL
```

Is the beginning of the GET_PRINCIPAL operation, for CID#26.


The operation ends with these messages:
```
(2025-06-02 16:46:01): [kcm] [kcm_cmd_done] (0x0400): [CID#26] KCM operation GET_PRINCIPAL returned [1432158224]: No matching credentials found
(2025-06-02 16:46:01): [kcm] [kcm_send_reply] (0x2000): [CID#26] Sending a reply
(2025-06-02 16:46:01): [kcm] [kcm_output_construct] (0x1000): [CID#26] Sending a reply with 4 bytes of payload
(2025-06-02 16:46:01): [kcm] [queue_removal_cb] (0x0200): [CID#26] Removed queue for 0
```
The first entry indicates that the GET_PRINCIPAL operation's result is "No matching credentials found."
The entry with the message "Sending a reply" states that the reply is being sent
to the client application, with a payload of 4 bytes as stated in the next entry.

The same operation can be requested several times during the same client session.
They are independent of each other.


### Automatic TGT Renewal

"Automatic TGT Renewal", also "TGT renewal" or "credential renewal", is dictated
by the value of the "tgt_renewal" option. When "tgt_renewal" is set to "true",
"1" or "yes", automatic credential renewal is enabled; when the option
"tgt_renewal" is set to "false", "0" or "no", tgt renewal is disabled.

When automatic tgt renewal is enabled, the "krb5_renew_interval" option can be
set to a time period expressed with units: s for seconds, m for minutes,
h for hours, d for days. A value of "3m" means 3 minutes or, in other words, 180
seconds. If a numeric value without unit is provided, it is assumed to be in
seconds: a value of "60" means 60 seconds.

Periodically, with the interval given by krb5_renew_interval, the renewal task
will be launched. This is an example:

```
(2025-06-17 10:33:06:897462): [kcm] [kcm_renew_all_tgts] (0x0400): Found [1] renewal entries. 
(2025-06-17 10:33:06:897475): [kcm] [kcm_renew_all_tgts] (0x0400): Checking ccache [1000] for creds to renew
(2025-06-17 10:33:06:897531): [kcm] [kcm_creds_check_times] (0x2000): Time not applicable
(2025-06-17 10:33:06:897542): [kcm] [kcm_creds_check_times] (0x2000): Time not applicable
(2025-06-17 10:33:06:897551): [kcm] [kcm_creds_check_times] (0x2000): Time not applicable
```

The message 'Found [1] renewal entries' means that one entry was found to be
renewable, and the message 'Time not applicable' means that it was too early to
renew it. TGT can be renewed in the second half of their lifetime.

-----
# Answering Questions about the SSSD Logs

This is something to have in mind when answering questions about SSSD's logs

* Be concise and give short answers.
* Unless explicitly requested, do not compare PIDs. They will always be different.
* Do not compare the timestamps. They will always be different. However, it is
possible to compare the duration of certain tasks. For instance, the duration
of two start-up phases.
* When asked to compare two commands, that means to compare the whole execution
of both commands (all the entries with the same CID), not just the entry with
the command name.


