---
title: WWDC 2017 Keynote Wishlist Followup
tags: apple WWDC
category: tech
---

The WWDC keynote came and went, and I thought I'd review here which of [my wishes]({% post_url 2017-06-01-wwdc-2017-wishlist %}) came true:

### Xcode
* Statically linked swift libraries: ❌
   - Though there are some WWDC sessions on how to improve dylib loading time, which might be enough.
* No more need for the simulator to run application unit tests: ❌
* Better Refactoring (renaming, extract method, etc) ✅
* More Xcode extension options: ❌
* Better swift migrator: ✅
   - I've run this on one moderately-sized OSS project so far, and the process seems very smooth compared to the 2 to 3 migrator. Time will tell how much better it is for very large codebases but I'm cautiously optimistic that this will be a win.

### iOS
* Better inter-app communication: ✅
   - Drag & Drop looks awesome for this, and the document improvements along with the new files app will be really nice.
* Allow non-webkit browser rendering engines for browsers: ❌
   - I didn't expect this but I can dream!
* Allow users to completely turn off internet access per-app: ❌

### Deployment
* An officially supported, unrestricted way to sideload apps outside of the app store: ❌
   - Didn't expect this one either 🤷‍♂️
* If we can't get that, then less restrictions on app provisioning: ✅
   - I haven't read into the details but apparently testflight is going to have fewer restrictions which is nice. I need to look into how they'll shape up though. Xcode 9 also has more fine-grained configuration options for controlling provisioning/signing which is a welcome addition, so I'll call this one a wish that came true.

### Final Score: 4/10

Not too shabby, though it would have been awesome to get even more.

The Xcode 9 improvements are also very welcome. I love the source editor changes for warnings/alerts, the new speedy indexing is awesome, and the github/better git integration is welcome. I wish I could start using this day to day! 😄
