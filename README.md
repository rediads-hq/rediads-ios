# RediadsSDK for iOS

Banner and interstitial ads from the Rediads ad server. RediadsSDK is a thin layer over the Prebid
Mobile SDK (our fork of it): it configures Prebid for our server and exposes a small API, so
apps never deal with Prebid, OpenRTB or stored requests directly.

Requires iOS 15 or later.

## 1. Before you start

Ask Rediads for an **account ID**, and have your app's **bundle ID** registered for it. The ad
server rejects apps whose bundle ID is not registered.

## 2. Add the package

Pick **one** product, the one for how your app shows ads; everything it needs (the Rediads core and
engine) comes with it.

| Product | Gives you | Import |
|---|---|---|
| **RediadsDisplay** | Banners, interstitials and rewarded ads rendered by the SDK | `import RediadsDisplay` |
| **RediadsGAM** | Google Ad Manager: bidding-only (GAM renders) or SDK-rendered (GAM decides, the SDK renders our wins) | `import RediadsGAM` |
| **RediadsAdMob** | AdMob mediation: Rediads competes as a custom event, the SDK renders our wins | `import RediadsAdMob` |

Also add **RediadsOM**, the Open Measurement SDK (viewability measurement for advertisers and
verification companies such as IAS and DoubleVerify). It has no code to import; the SDK finds it
while the app runs. Leave it out only when the app already has Prebid Mobile (see below).

**Swift Package Manager.** In Xcode: **File > Add Package Dependencies**, enter
`https://github.com/rediads-hq/rediads-ios.git`, keep **Up to Next Major Version** and tick the
product. In a `Package.swift`:

```swift
.package(url: "https://github.com/rediads-hq/rediads-ios.git", from: "0.1.2"),
// and in your target's dependencies:
.product(name: "RediadsDisplay", package: "rediads-ios"),
```

**CocoaPods.** The pods have the same names and imports as the products. Each pod's podspec comes
from the repository at the version's tag, so list every Rediads pod your product needs: the first
four always, plus the pair for GAM or AdMob if you use it.

```ruby
rediads = 'https://raw.githubusercontent.com/rediads-hq/rediads-ios/0.1.2'
pod 'RediadsEngine', :podspec => "#{rediads}/RediadsEngine.podspec"
pod 'RediadsCore', :podspec => "#{rediads}/RediadsCore.podspec"
pod 'RediadsDisplay', :podspec => "#{rediads}/RediadsDisplay.podspec"
pod 'RediadsOM', :podspec => "#{rediads}/RediadsOM.podspec"
# Google Ad Manager (RediadsGAM): also these two
pod 'RediadsGAMEventHandlers', :podspec => "#{rediads}/RediadsGAMEventHandlers.podspec"
pod 'RediadsGAM', :podspec => "#{rediads}/RediadsGAM.podspec"
# AdMob mediation (RediadsAdMob): also these two
pod 'RediadsAdMobAdapters', :podspec => "#{rediads}/RediadsAdMobAdapters.podspec"
pod 'RediadsAdMob', :podspec => "#{rediads}/RediadsAdMob.podspec"
```

Google's pods declare iOS 12, which current Xcode rejects ("deployment target ... is set to 12.0").
Raise the pods to 15.0 at the end of the Podfile:

```ruby
post_install do |installer|
  installer.pods_project.targets.each do |target|
    target.build_configurations.each do |config|
      config.build_settings['IPHONEOS_DEPLOYMENT_TARGET'] = '15.0'
    end
  end
end
```

**Objective-C.** Import the core module as well: Objective-C doesn't see the Swift module's
re-export of it.

```objc
@import RediadsCore;     // RediadsAds, RediadsAdEvent, RediadsReward, RediadsErrorCode
@import RediadsDisplay;  // or RediadsGAM, or RediadsAdMob

[RediadsAds startWithAccountId:@"YOUR_ACCOUNT_ID" serverURL:RediadsAds.defaultServerURL completion:nil];

RediadsBannerView *banner = [[RediadsBannerView alloc] initWithPlacement:@"home_banner"
                                                                    size:RediadsBannerView.banner320x50
                                                                  format:RediadsAdFormatDisplay];
banner.onEvent = ^(RediadsAdEvent *event) { NSLog(@"%@", event); };
[banner load];
```

The names follow the Swift ones (`loadWithCompletion:`, `showFrom:`, `loadBannerView:request:completion:`,
`loadWithGAMAdUnitID:request:bidCompletion:completion:`). Objective-C has no default arguments, so
pass every one. Compare errors with `RediadsAds.errorDomain` and `RediadsErrorCode...` values.

**Apps that also use Prebid Mobile.** The Rediads engine is our fork of Prebid Mobile under its own
names (module `RediadsEngine`, Objective-C classes prefixed `RDA`), so the two don't clash. Both use
the same Open Measurement framework, `OMSDK_Prebidorg.framework`, and Xcode allows only one copy in
an app ("Multiple commands produce …/OMSDK_Prebidorg.framework"). So **don't add RediadsOM** (or the
`RediadsOM` pod) to an app that has Prebid Mobile: Rediads uses Prebid's copy.

**Without any OM SDK** the ads still work, but without viewability measurement, and bid requests
don't offer it (no `7` in `api`, no OM partner), so buyers that require measurement won't bid. The
SDK logs a warning at start: "Open Measurement SDK not found".

## 3. Start the SDK once, at launch

```swift
import RediadsDisplay

func application(_ application: UIApplication,
                 didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
    RediadsAds.start(accountId: "YOUR_ACCOUNT_ID")
    return true
}
```

Options, set before `start`:

| Setting | Default | Use |
|---|---|---|
| `RediadsAds.testMode` | `false` | Request test bids while integrating. Never ship it on. |
| `RediadsAds.debugLogging` | `false` | Detailed SDK logs in the console. |
| `RediadsAds.childDirected` | `false` | Turn on if your app is aimed at children under 13 (US COPPA). Can change at any time, e.g. per user in apps for mixed audiences. See Privacy. |
| `RediadsAds.appStoreID` | `nil` | Your app's App Store ID, the number in its App Store link (`"123456789"` for `apps.apple.com/app/id123456789`). Ad networks need it for app-install ads that open the App Store inside your app. Set it once your app is on the App Store. |
| `serverURL:` argument | production | Only for testing against another server. |

## 4. Banner

```swift
let banner = RediadsBannerView(placement: "home_banner", size: RediadsBannerView.banner320x50)
banner.onEvent = { event in
    print(event)                       // loaded, displayed, failed, clicked, ...
}
container.addSubview(banner)
banner.load()
```

- `format:` (optional) is a `RediadsAdFormat`, the same type interstitials and rewarded ads use:
  `.display` (default), `.video` (a video plays inside the banner; use it for 300x250 and larger) or
  `.displayAndVideo`. Video in banners is in testing.
- `placement` names the slot; it appears as the slot in Rediads reporting. Use one name per ad
  position.
- Sizes: `banner320x50`, `mediumRectangle300x250`, `leaderboard728x90`, or any `CGSize`.
- The banner refreshes every `refreshInterval` seconds (15 to 120, default 60) while on screen.
  Call `stopRefresh()` to stop.
- Keep a strong reference to the banner (for example as a property, or through its superview).

## 5. Interstitial

```swift
var interstitial: RediadsInterstitial?

func prepareAd() {
    let ad = RediadsInterstitial(placement: "level_end")       // or format: .video, .displayAndVideo
    ad.load { error in
        if let error { print("No interstitial: \(error.localizedDescription)") }
    }
    interstitial = ad
}

func levelFinished() {
    if interstitial?.isReady == true {
        interstitial?.show(from: self)
    }
    prepareAd()                                               // one ad per instance
}
```

Load ahead of time and show at a natural break. Each instance shows one ad.

## 5a. Rewarded

```swift
var rewarded: RediadsRewarded?

func prepareRewarded() {
    let ad = RediadsRewarded(placement: "extra_life", reward: RediadsReward(type: "lives", amount: 1))
    ad.onEvent = { event in
        if event.type == .rewarded, let reward = event.reward {
            grant(reward)                                     // your own reward logic
        }
    }
    ad.load()
    rewarded = ad
}

func userAskedForExtraLife() {
    if rewarded?.isReady == true {
        rewarded?.show(from: self)
    }
}
```

You define the reward; the `rewarded` event hands it back, once per ad. Display ads grant it
after `displaySeconds` on screen (an `init` argument: default 15, allowed 5 to 120), video ads (`format: .video` or `.displayAndVideo`) when playback
completes. The close button appears once the reward is granted. Create a new instance for the next
ad.

## 5b. Google Ad Manager (bidding-only)

GAM classes come in two kinds. `…BiddingOnly` classes add the Rediads bid to a GAM ad you own, and
GAM renders. The others (`RediadsGAMBannerView`, `RediadsGAMInterstitial`) are ad objects like
`RediadsBannerView` and `RediadsInterstitial`: GAM decides, the Rediads SDK renders our wins.

If GAM is your ad server, keep your GAM banner and let Rediads add its bid to each request. GAM
runs its auction; when a Rediads line item wins, GAM shows the Prebid Universal Creative, which
loads the ad from the Rediads cache. Add the **RediadsGAM** product, plus Google's
`GoogleMobileAds` package.

```swift
import GoogleMobileAds
import RediadsGAM

// At launch: RediadsAds.start(accountId:) and MobileAds.shared.start()

let gamBanner = AdManagerBannerView(adSize: AdSizeMediumRectangle)
gamBanner.adUnitID = "/NETWORK_CODE/your_ad_unit"
gamBanner.rootViewController = self
gamBanner.delegate = self            // GAM reports what it rendered

let rediads = RediadsGAMBannerBiddingOnly(placement: "your_ad_unit", size: RediadsGAMBannerBiddingOnly.mediumRectangle300x250)
rediads.load(gamBanner)              // fetches our bid, adds hb_ keys, loads GAM
```

- GAM is always loaded, with or without a Rediads bid. The optional completion says whether a
  Rediads bid was added.
- `fetchTargeting(completion:)` returns the `hb_` keys only, if you build the GAM request yourself.
- Call `load` again to refresh.
- `load` puts a stand-in in `gamBanner.delegate` that passes every callback on to your delegate and
  reports what GAM served to Rediads telemetry. Set your delegate before `load`; one set later is
  picked up at the next `load`. Reading `gamBanner.delegate` returns the stand-in.
- Add `format: .video` (300x250 and larger) or `.displayAndVideo` for video; GAM then plays our VAST
  in Google's player. Video needs its own line items (`hb_format is video`, VAST redirect creative).
- GAM setup needed: line items targeting `hb_pb` (and `hb_env = mobile-app`) with the Prebid
  Universal Creative, and the app's Google Mobile Ads app ID in `Info.plist`
  (`GADApplicationIdentifier`).
- Full GAM setup (key-values, creative, line items, testing, troubleshooting): ask Rediads for
  the GAM setup guide.

## 5c. Google Ad Manager (SDK-rendered)

GAM still decides, but when the Rediads line item wins, its creative only fires an app event and the
Rediads SDK renders the ad itself (with Rediads trackers, MRAID and OM). The SDK owns the view:

```swift
let banner = RediadsGAMBannerView(placement: "home_mrec",
                                  gamAdUnitID: "/NETWORK_CODE/home_mrec",
                                  size: RediadsGAMBannerView.mediumRectangle300x250)
container.addSubview(banner)
banner.load()                        // banner.lastRenderer says who rendered: .rediads or .gam
```

Add `format: .video` or `.displayAndVideo` to let Rediads video compete. Needs a `rediads_render`
key-value and its own line items with an app-event creative. Ask Rediads
for the setup guide.

### GAM interstitials

| Path | Class | You get |
|---|---|---|
| Bidding-only | `RediadsGAMInterstitialBiddingOnly` | Google's `AdManagerInterstitialAd` with our bid in the request; you present it |
| SDK-rendered | `RediadsGAMInterstitial` | One object: `load`, `isReady`, `show(from:)`, events for either winner |

Both interstitial classes take `format:` for video. Rewarded through GAM: `RediadsGAMRewarded`
(SDK-rendered, same shape plus your `reward`) or `RediadsGAMRewardedBiddingOnly` (video only; you get
Google's `RewardedAd` and GAM grants the GAM ad unit's reward).

Details in the two GAM guides.

## 5d. AdMob mediation

AdMob decides; Rediads is a custom event in the app's AdMob mediation group, and when it wins the
Rediads SDK renders the ad inside AdMob's banner or full-screen ad. You keep AdMob's objects:

```swift
let banner = RediadsAdMobBanner(placement: "home_mrec", bannerView: adMobBanner)   // your AdMob BannerView
banner.load()

let rewarded = RediadsAdMobRewarded(placement: "extra_life")
rewarded.load(adMobAdUnitID: "ca-app-pub-XXXX/YYYY") { ad, error in self.adMobRewarded = ad }
```

`RediadsAdMobInterstitial` works like the rewarded one. AdMob setup (app, ad units, mediation group
with the custom event class names and `{"hb_pb":"5.00"}` parameters): ask Rediads for the AdMob setup guide.

## 6. Events and errors

`onEvent` delivers a `RediadsAdEvent` on the main thread: `type` (`loaded`, `displayed`,
`failed`, `clicked`, `presented`, `dismissed`, `leftApplication`, `rewarded`), `placement`,
`adSize`, `error` for failures and `reward` for `rewarded`.

Errors are `NSError`s in `RediadsError.domain` (`RediadsAds.errorDomain` in Objective-C) with a
`RediadsErrorCode`:

| Code | Meaning | What to do |
|---|---|---|
| `noFill` | No bidder wanted the impression | Normal. Try again later. |
| `timeout` | No response within 1 second | Try again later. |
| `network` | Server unreachable | Check connectivity. |
| `invalidConfiguration` | Bad account, app not registered, bad size | Fix the integration. |
| `notStarted` | `start` was not called | Call `RediadsAds.start` first. |
| `notReady` | `show` before the interstitial or rewarded ad loaded | Check `isReady`. |
| `renderFailed`, `server`, `unknown` | Other failures | The original error is in `NSUnderlyingErrorKey`. |

## 7. Privacy

- Add your SKAdNetwork IDs and an App Tracking Transparency usage message
  (`NSUserTrackingUsageDescription`) to `Info.plist` if you request tracking permission. With
  SKAdNetwork IDs, also set `RediadsAds.appStoreID`: the IDs and your App Store ID go with every
  ad request.
- If you use a consent management platform (CMP), the SDK reads its standard IAB consent strings
  automatically.
- **Apps for children:** set `RediadsAds.childDirected = true`. Ad requests are then marked as
  directed to children (COPPA), and the Rediads server removes device IDs, the IP address, location
  and user data before any bidder sees the request. In apps for mixed audiences, turn it on while
  the user is known to be under 13. It applies from the next ad request.
- **Privacy manifest:** the SDK ships one (`PrivacyInfo.xcprivacy`, in the `RediadsCore` bundle).
  Xcode merges it into your app's privacy report (*Product → Archive*, then *Generate Privacy
  Report* in the Organizer); use that report for your App Store privacy details. It declares:
  device ID (IDFA when the user allows tracking, otherwise IDFV; used for tracking), advertising data
  (ads shown), product interaction (ad clicks) and coarse location (from the IP address), all for
  third-party advertising and analytics; performance data and other diagnostic data (telemetry, not
  linked to the user), for app functionality and analytics; and UserDefaults (consent strings, the
  SDK's own cache). If you turn on Prebid's `shareGeoLocation`, also declare precise location
  yourself.
- **Telemetry:** the SDK sends Rediads diagnostics so we can find problems your users run into: how
  long each ad request and each ad load took, why one failed (timeout, network, server, an ad that
  couldn't render), for GAM bidding-only ads what GAM served (line item and creative IDs), ad
  clicks, the ad type and placement, and the SDK, app and OS versions and device model.
  No user or device identifiers. Events are batched (at most every 30 s, and when the app goes to
  the background). To turn it off, set `RediadsAds.telemetryEnabled = false` before `start`.

