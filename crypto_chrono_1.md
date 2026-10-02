Chrono I — Writeup

Challenge: ESUITO{gwfvb_xejqnf_nimgt_b_whhrlv}

Step 1: Spot the key hint

The challenge name "Chrono" plus a timestamp-looking string hints that the key is a date/time value used as a numeric keystream (classic Vigenère-with-digits style cipher).

Stripping the separators (/, space, :) from the timestamp gives a 14-digit key:

20260921143507
Step 2: Set up the cipher

Treat letters as a=0, b=1, ..., z=25, and subtract the key digits (cycled repeatedly) from each ciphertext letter's value, mod 26. Non-letters (_, {, }) pass through unchanged and don't consume a key digit.

Key digit stream (indexed):

idx: 0  1  2  3  4  5  6  7  8  9 10 11 12 13
val: 2  0  2  6  0  9  2  1  1  4  3  5  0  7
Step 3: Decrypt the header
Cipher	Value	Key	    Calc	    Plain
E	    4	    2	    4−2=2	    C
S	    18	    0	    18−0=18	    S
U	    20	    2	    20−2=18	    S
I	    8	    6   	8−6=2	    C
T	    19	    0	    19−0=19	    T
O	    14	    9	    14−9=5	    F

→ CSSCTF confirms the key and shift direction are correct.

Step 4: Decrypt the body (continuing the key stream from index 6)

Letters only consume key digits; underscores are skipped but preserved:

Chunk	Key indices used	                Result
gwfvb	6,7,8,9,10 → (2,1,1,4,3)	        every
xejqnf	11,12,13,0,1,2 → (5,0,7,2,0,2)	    second
nimgt	3,4,5,6,7 → (6,0,9,2,1)	            hides

b	8 → (1)	a
whhrlv	9,10,11,12,13,0 → (4,3,5,0,7,2)	secret

Final Flag
CSSCTF{every_second_hides_a_secret}