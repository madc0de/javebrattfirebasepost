# Learn how to build End-to-End Encrypted (HIPAA-compliant) chat with Firebase using open source libraries

Firebase is an awesomely productive way to build the “necessary backend stuff” for your app. Rumor says that one of the first Firebase samples (before the 2014 Google acquisition) was a chat app, which drove tons of developers to built in-app chat using the platform. And around the same time, healthcare app developers’ demand for Firebase’s HIPAA compliance starting piling up, which remains a demand today, still.

Since then, End-to-End Encrypted chat apps surfaced and got onto our phones. End-to-End Encryption, which once was the luxury of a few companies (WhatsApp/Facebook, Microsoft’s Skype) is now a tech that developers can build into their apps in days. As we learned from FBI & Apple’s case and by seeing how governments of the world react to chat apps that prohibit public surveillance, End-to-End Encryption is the security that’s proven to be the strongest to resist data breaches, hacks, curiosity. And it’s the one that enables healthcare developers to design their apps in a way that their cloud providers (such as Firebase) don’t see their users’ health data, which makes the monster of HIPAA compliance ease up a bit.

In this article, I’ll walk you through the design decisions in building an End-to-End Encrypted (and HIPAA-compliant) Firebase chat app.

> If you want to cut the chase, download the [iOS](https://github.com/VirgilSecurity/demo-firebase-ios) and [Android](https://github.com/VirgilSecurity/demo-firebase-ios) samples from GitHub and start playing with them right away.

## What is End-to-End Encryption?

First, let’s start with a quick refresher of what end-to-end encryption (“E2EE”) is and how it works.

This is what your Firebase app looks like today:

![Your data in Firebase](Firebase_security.png)

Every component in your backend (i.e. Firebase/Google’s backend) has access to every single bit of data that your apps submit to your backend services. No secrets. No privacy. High risk of data breach. And definitely not HIPAA-compliant.

And this is what your app will look like after you implement client-side end-to-end encryption:

![Your data in Firebase](FirebaseE2EE.png)

The messages are encrypted on the user device and remain encrypted as they travel over the mobile network/Wi-Fi/Internet, through the cloud/web server, into Firestore, and on the way back to your chat partner (such as a patient or doctor in a healthcare scenario). In other words, none of the networks or servers will have a clue what the two of you are chatting about. It’s the WhatsApp way.

![Your data in Firebase](Firebase_Firestore.png)

## What can I end-to-end encrypt?

Anything. Really. Chat messages, file or data transfers, even live streams. In this example, we keep our focus on In-app Chat

## Can I just do it myself from scratch?

Yes! What’s difficult in end-to-end encryption is somehow managing the encryption keys in a way that only the users involved in the chat can access them and nobody else. And when we say “nobody else,” we really mean it; even your cloud provider or you, the developer, are out; no accidental mistakes or government-ordered peeking are possible.

Writing crypto, especially to work across multiple client platforms (iOS, Android, JS), is hard. Generating truly random numbers, picking the right algorithms for the operating system/browser, and choosing the right encryption modes are just a few examples of why most developers wave the white flag and end up NOT doing it. It famously took WhatsApp 3 years and 15 smart developers to make their chat app end-to-end encrypted, on par with project timelines at Facebook Messenger, Apple and other tech companies.

To make the implementation simpler, we use the open source Virgil SDK which provides a cross-platform crypto library and key management service to store public keys.

## Okay, how do I end-to-end encrypt my Firebase chat app?

In a nutshell (and in Swift):
 
1. For every new user, the app generates a private & public key in your app (on the client device):

    ```
    let keyPair = try self.crypto.generateKeyPair()
    ```

2. The public key is published to our Cards Service, so that other users of your app can retrieve it to encrypt data for you:

    ```
    cardManager.publishCard(privateKey: keyPair.privateKey,
            publicKey: keyPair.publicKey,
            identity: identity) 
    ```

	Before you’d ask – no, we don’t send up the private key to the server.

3. The private key remains on your device and lives in your mobile operating system’s native key store:

    ```
	try self.keyStorage.store(privateKey: keyPair.privateKey, 
            name: identity, meta: nil)
    ```
    
4. Before sending a chat message, the app encrypts it with the recipient’s public key:

	```
	return try self.crypto.encrypt(data, 
            for: self.channelKeys + self.selfKeys).base64EncodedString()
    ```
    
5. After receiving a message, the app decrypts it with your user’s private key, which never leaves the device:

    ```
	let decryptedData = try self.crypto.decrypt(data, 
            with: privateKey)
    ```
    
The technique above ensures that only end-users are able to see messages; the data won’t be visible to Firebase, Google or developers/IT staff at your company. It’s now an end-to-end encrypted chat, like your WhatsApp conversations.

## What if I’m building a group chat?

If you’re implementing group chat and you want to avoid encrypting all messages with all participating users’ public keys, and then client-side re-encrypting them when a new user joins the conversation, do this:

1. Create a new symmetric key pair (generateKey, i.e. the same key is used for encryption & decryption) for every new chat thread,

2. Use that key to encrypt/decrypt all messages in the thread. It’s symmetric to prevent non-participating users from being able to write messages to the thread.

3. Encrypt the key with all participating users’ public keys and store the encrypted key in Firestore (which is fine, because it can’t be decrypted without the users’ private keys). This way, you cryptographically share the thread key with all users in the the thread. [Ping us on Slack](https://join.slack.com/t/virgilsecurity/shared_invite/enQtMjg4MDE4ODM3ODA4LTc2OWQwOTQ3YjNhNTQ0ZjJiZDc2NjkzYjYxNTI0YzhmNTY2ZDliMGJjYWQ5YmZiOGU5ZWEzNmJiMWZhYWVmYTM) if you have any questions or need help implementing this.

## What happens if my user loses her device with the private key on it?

As in the sample, your messages are local-only (Firebase is only an intermediate delivery system), so the messages will be lost with the phone. However, if there’s (non-patient) data that you want to permanently store encrypted in Firebase, there’s a solution for lost devices/keys.

Drumroll, please… Imagine a private key that can be lost, and you don't have to worry about losing your encrypted data. Or, imagine a private key that you can use between devices without needing to transfer the key around (WOW!). This special key is called “BrainKey”: a strong cryptographic key that’s based on your user’s PASSWORD. Every time you capture your user’s password, you can re-generate the same private key, so it won’t be lost with the device:

```
let brainKeyContext = BrainKeyContext.makeContext(
        accessTokenProvider: accessTokenProvider)

let brainKey = BrainKey(context: brainKeyContext)

// Generate default public/private Curve ED25519 keypair
let keyPair = try! brainKey.generateKeyPair(
        password: "Your password", 
        brainKeyId: "Your id here").startSync().getResult()
```

Here’s how to implement your user private key recovery:

1. At signup, generate a BrainKey right after you generate your user’s private key.
2. Encrypt the user’s private key with her BrainKey’s public key.
3. Store the encrypted private key in your Firestore where you store the user’s profile data.
4. At login-time, re-generate the BrainKey from her password.
5. Download the encrypted user private key from Firestore.
6. Decrypt it with the BrainKey, and Voila! You have a way to recover the key if a device is lost.
 
> How to deal with a forgotten password and lost device at the same time? Use a concatenated string of 3 secret answers as a password to generate a secondary BrainKey for your users. Use this secondary BrainKey in the same way as you do the password BrainKey: encrypt, then later “unlock” the user private key. Now you have a user lockout solution with 3 secret questions+answers!

## What about HIPAA for healthcare apps?

In building the samples, we took the approach that worked for Twilio: it doesn’t require a [HIPAA Business Associate Agreement](https://www.hhs.gov/hipaa/for-professionals/covered-entities/sample-business-associate-agreement-provisions/index.html) (a “BAA” which means you assume the liability for the security of the personal data). We did this by preventing you developer and any of your cloud/service providers from seeing or storing any data - even encrypted health data - beyond the message delivery. Then we built these techniques into the iOS and Android sample apps (JS coming soon) and audited the approach and implementation with RedLion.io, experts in InfoSec & HIPAA compliance.

> To learn more about how the samples at the end of this article meet HIPAA requirements, check out the [Firebase Chat HIPAA whitepaper](https://virgilsecurity.com/wp-content/uploads/2018/07/Firebase-HIPAA-Chat-Whitepaper-Virgil-Security.pdf). If you have any questions, [join our Slack channel](https://join.slack.com/t/virgilsecurity/shared_invite/enQtMjg4MDE4ODM3ODA4LTc2OWQwOTQ3YjNhNTQ0ZjJiZDc2NjkzYjYxNTI0YzhmNTY2ZDliMGJjYWQ5YmZiOGU5ZWEzNmJiMWZhYWVmYTM) and chat with us!

## Get your hands on the app

Download the [iOS](https://github.com/VirgilSecurity/demo-firebase-ios) and [Android](https://github.com/VirgilSecurity/demo-firebase-ios) samples from GitHub and start playing with them!

David Szabo, VP of Developer Platform at Virgil Security, San Francisco
