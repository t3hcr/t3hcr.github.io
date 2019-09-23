*Posting this here, just incase anyone else is trying to move the future along and dealing with this bug.*

# Case/issue: 
When activating Windows Server 2019 via the GUI and with a MAK key, you receive the following message: "The product key you entered didn't work. Check the product key and try again, or enter a different one."  (in my case it listed error code 0x80070490 - your error may vary)

# How I resolved the issue: 
Via Admin-elevated Command Prompt, I entered the following: `slmgr /ipk <INSERT-YOUR-MAK-KEY-HERE>`

# Scratching my head
I'm a bit confused as to why something like this exists in a modern server OS, but hey - at least it's not a huge roadblock.  While I prefer to keep activations with MAK keys to a minimum (and use KMS), in this instance a MAK was necessary.  (a non-domain joined host for a specific purpose)

Hope this helps someone today or further into the future.