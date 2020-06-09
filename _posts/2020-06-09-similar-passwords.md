---
title: An approach to preventing the use of similar passwords when enforcing change
---

# Introduction

Forcing users to change their password on a regular basis has been a staple of authentication design. [Recent thoughts ](https://www.troyhunt.com/passwords-evolved-authentication-guidance-for-the-modern-era/) on the matter have questioned the necessity of forcing users to change passwords, especially in the context of better password managers. These can generate very strong passwords on an app by app basis mitigating much of the problems related to passwords.

Still, there are occasions where enforcing a password change is necessary. For example, a data breach may require the resetting of passwords for a selection of users.

I recently worked on a project where we wanted to mandate a password change but also prevent the re-use of existng passwords or ones similar to those.

# Storing Passwords

There's a lot of existing literature on best practice for storing passwords so I won't repeat that in detail. The key things to remember are:

* Don't store the actual password
* Use a salt plus hashing algorithm to store text that can be compared with the known password but does not allow the password to be recovered
* Use a sufficiently complex hashing algorithm to slow down brute force attempts

The problem that then arises is, if we are unable to store the password, how do we know whether the users new password is similar to the old one? Worse, a good hashing algorithm for this use case should ensure that similar passwords result in very different hashes.

# Password Similarity

Having read through the requirements for password storage it seems that checking password similarity is difficult to do. 

We took inspiration from full text search algorithms and devised an approach similar to [stemming ](https://en.wikipedia.org/wiki/Stemming). Stemming, in the linguistic context, is the reduction of words to a common form so that similar words can all match a search query. For example, a stemming algorithm might reduce the words fishing, fished, and fisher to the stem fish.

Clearly the problem with these general purpose stemming algorithms is that they work on written text, and passwords are not that! 

If you think about the format of passwords there are lots of different patterns that people use to generate them:

- a mix of words
- some words with digits added on
- a complete mix of letters and symbols
- words with 1's swapped out for !
- password123 

Our first step is to prevent users being able to use the top 10,000 most common passwords i.e. no matter how disimilar password123 is from a previous password we're not going to let them use it.

The next thing to notice is that, a lot of the passwords that people use are actually words they know well but manipulate slightly to "make them in to passwords". 

What we do is make use of this insight to generate possible permutations of a password based on common substitutions and to then extract that password in tokens of known words and other things. We then make use of a standard stemming algorithm to derive a stemmed version of the password. We then store this stemmed version of the password in exactly the same way that we store the actual password.

When the user attempts to change the password we run the same algorithm, stemming their new password, and then compare it's hash to all of their previous hashes.

One question arises: 

> Does storing the stem of a password make it more likely that the actual password will be recovered?

There is indeed a slight increase in the probability of recovery as an attacker can now check stemmed passwords in order to narrow down the actual password quicker. However, the possibility is still low as the stemmed password is hashed using a salt in the same way the password is. 

