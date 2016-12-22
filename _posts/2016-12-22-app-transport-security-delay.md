---
title: Apple Delays App Transport Security Deadline
category: security
tags: app-store app-transport-security
---

Apple is [delaying their previously announced enforcement of App Transport Security](https://developer.apple.com/news/?id=12212016b)  until further notice:

> App Transport Security (ATS), introduced in iOS 9 and OS X v10.11, improves user security and privacy by requiring apps to use secure network connections over HTTPS. At WWDC 2016 we announced that apps submitted to the App Store will be required to support ATS at the end of the year. To give you additional time to prepare, this deadline has been extended and we will provide another update when a new deadline is confirmed. [Learn more about ATS.](https://developer.apple.com/videos/play/wwdc2016/706/)

I was very encouraged when they first introduced ATS, since as an iOS user I knew that the requirements it imposed would have a positive impact on the security of the apps many people use each day. Hearing that Apple would enforce ATS with only some case-by-case exceptions as part of app review was even better. As a security-conscious individual, reading about this delay is disheartening to say the least.

I can only speculate on why Apple may be doing this. My first guess is that they're recording how many apps are still being submitted with ATS exceptions, and that amount must be too great for them to feel comfortable enforcing this new rule. They may not want the hassle of manually approving apps that ship with ATS exceptions, or they may not want to lock developers out of providing fixes and updates to users if the security of the services those apps use largely aren't in developers' control.

If you haven't yet, I recommend reading the [guide to configuring ATS in your app](https://developer.apple.com/library/content/documentation/General/Reference/InfoPlistKeyReference/Articles/CocoaKeys.html#//apple_ref/doc/uid/TP40009251-SW33) which is pretty comprehensive. One thing it doesn't note though is that (as of iOS 9, last time I checked) there's also a requirement that the TLS certificate have a SHA256 fingerprint, which is not documented here (that requirement is wrapped in the perfect forward secrecy exception). If everything else according to your certificate is up to snuff, this might be the sticking point if your server has an old cert still installed. Certificate authorities have stopped issuing new certs that don't use SHA256 as of January 2016, so this most likely won't happen now, but it's something to watch out for.

You can also check if the services you connect to are ATS-ready in a couple of simple ways:

1. Using the [ssl test at SSL Labs](https://www.ssllabs.com/ssltest/). If you get a grade of A or better, you're almost certainly good to go. You'll get a great report on what your certificate and server support for TLS, and any major problems are explained clearly and in detail.
2. Using `nscurl` on the command line from macOS. I haven't explored what else `nscurl` does, but it provides a handy tool that you can use to see which ATS configurations your server will pass and fail with: `nscurl --ats-diagnostics https://www.example.com`. You can then learn what the smallest set of exceptions you need in order to configure ATS for your app are.

I strongly urge you to do whatever you can to influence the security of the services you consume. If your company doesn't yet support the most recent versions of TLS with strong ciphers that use perfect forward secrecy and certificates created this year, pushing for those changes can only help protect your users from having their data stolen. If the services you use are external, then you have less influence, but any pressure you can put on those providers to employ modern security practices will help us all enjoy a more secure and private experience on the internet.
