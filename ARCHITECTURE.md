# Shared SwiftUI architecture

Most of the native apps in this portfolio share one small, dependency-free architecture. It lets a new client app start from a working shell, so the time goes into what makes each one different: the design system, the content and the one or two features that client actually needs.

The snippets below are trimmed from the apps. They contain no client data or credentials.

## 1. Project shell

- **iOS 17+, SwiftUI only, zero third-party packages.** No CocoaPods, no SPM dependencies, nothing to update or audit.
- **File-system-synchronized Xcode groups** (`PBXFileSystemSynchronizedRootGroup`, objectVersion 77). Any Swift file or asset dropped into the app folder is compiled automatically, so a new app starts as a copied `.xcodeproj` with the target renamed.
- **Fixed file layout** that every app follows:

```
App/
├── <Name>App.swift      @main, environment objects
├── DesignSystem.swift   palette, type scale, reusable modifiers (one per app)
├── Components.swift     cards, kickers, buttons, FillImage, video loops
├── AppData.swift        all static content: copy, links, schedules, ministries
├── Models.swift         Sermon, Event, Ministry, ...
├── SermonService.swift  YouTube feed loader
├── WebSheet.swift       in-app Safari + link policy
└── Views/               RootView (custom tab bar) + one file per tab
```

All copy, prices and links live in `AppData.swift` / `Models.swift`. A client edit is usually a one-file change.

## 2. One design system per app

Each app gets its own named design language: "Harvest", "Vesper", "Draft", "Marquee", "Petal" and so on. Views never use raw colors or font sizes. They read tokens from a single `DS` (or `Theme`) enum, so re-skinning an app means editing one file:

```swift
/// "Harvest": black canvas, harvest amber, sky-blue accents.
enum DS {
    static let ink   = Color(red: 0.020, green: 0.020, blue: 0.024)  // #050506
    static let amber = Color(red: 0.930, green: 0.550, blue: 0.110)  // #ED8C1C
    static let sky   = Color(red: 0.260, green: 0.580, blue: 0.860)  // #4294DB
    static let gold  = Color(red: 0.976, green: 0.741, blue: 0.328)  // #F9BD54

    static func display(_ size: CGFloat) -> Font {
        .system(size: size, weight: .heavy, design: .rounded)
    }
}
```

Apps that follow the system appearance define their tokens as **Color Set assets** with explicit light and dark variants, so dark mode is a deliberate design and not an automatic inversion. One lesson from that work: keep "adaptive accent text" and "text on a solid accent fill" as separate tokens from day one.

```swift
enum Theme {
    /// Adaptive: deep navy in light mode, soft light-blue in dark mode.
    static let navy = Color("Navy")
    /// Fixed deep navy in both modes, only for text sitting on a solid gold fill.
    static let navyFixed = Color("NavyFixed")
    static let gold = Color("Gold")
    static let background = Color("Background")
}
```

## 3. Sermons from YouTube without an API key

Churches already publish every sermon to YouTube. The apps read the channel's public Atom feed (`https://www.youtube.com/feeds/videos.xml?channel_id=...`) with Foundation's `XMLParser`. That means no API key to ship or rotate, no quota and no backend.

```swift
@MainActor
final class SermonService: ObservableObject {
    enum State { case loading, loaded([Sermon]), failed }
    @Published var state: State = .loading

    func load() async {
        state = .loading
        do {
            let (data, _) = try await URLSession.shared.data(from: AppData.sermonFeedURL)
            let sermons = FeedParser().parse(data)   // XMLParserDelegate over <entry>
            state = sermons.isEmpty ? .failed : .loaded(sermons)
        } catch {
            state = .failed
        }
    }
}
```

Every channel titles its uploads differently, so each app's `Sermon` model parses the title format that channel actually uses: `Title | Speaker | Church`, `Title // Series // Speaker`, or `"Title" II Speaker Date`. It splits on the separator, drops the church-name segment and treats a segment that starts with a title (Pastor, Dr, Elder) as the speaker. Uploads that don't match the format, such as clips and `#shorts`, go to a separate Shorts rail instead of the sermon list:

```swift
struct Sermon: Identifiable, Codable {
    let id: String          // YouTube video ID
    let title: String

    private var parts: [String] {
        title.components(separatedBy: "|")
            .map { $0.trimmingCharacters(in: .whitespaces) }
            .filter { !$0.isEmpty && !$0.lowercased().contains(AppData.channelTag) }
    }
    var messageTitle: String { parts.first ?? title }
    var speaker: String? {
        parts.dropFirst().first { $0.hasPrefix("Pastor") || $0.hasPrefix("Dr") || $0.hasPrefix("Elder") }
    }
    var isFullMessage: Bool { parts.count >= 2 }
    var thumbnailURL: URL { URL(string: "https://i.ytimg.com/vi/\(id)/hqdefault.jpg")! }
}
```

A second generation of apps reads the channel's `/streams` page instead, so the list shows past livestreams rather than every upload. Those apps cache the last good result and ship a set of bundled seed videos, so the Messages tab is never empty offline.

## 4. "Next service" countdown

The home screen counts down to whichever gathering comes next and switches to a LIVE state during a service window. The schedule is plain data and always uses the church's own time zone, not the phone's:

```swift
enum ServiceSchedule {
    static let timeZone = TimeZone(identifier: "America/New_York")!

    private struct Window { let weekday, startHour, startMinute, endHour, endMinute: Int; let label: String }
    private static let windows: [Window] = [
        Window(weekday: 7, startHour: 10, startMinute: 0, endHour: 11, endMinute: 30, label: "Sabbath School"),
        Window(weekday: 4, startHour: 19, startMinute: 0, endHour: 20, endMinute: 0,  label: "Midweek Service"),
    ]

    static func nextService(now: Date = .now) -> (date: Date, label: String) {
        // Calendar.nextDate(after:matching:) for each window, pick the earliest.
    }
}
```

## 5. One sheet per screen, one link policy

SwiftUI honors only one `.sheet` modifier per view. Stacking several silently drops all but one, which shows up as buttons that do nothing. Each screen owns a single optional `Modal` enum and presents everything through it:

```swift
enum Modal: Identifiable {
    case web(URL), planVisit, about, notes(Sermon)
    var id: String { /* stable per case */ }
}

extension View {
    func modalSheet(_ modal: Binding<Modal?>) -> some View {
        sheet(item: modal) { presented in
            switch presented {
            case .web(let url):      WebSheet(url: url).ignoresSafeArea()  // SFSafariViewController
            case .planVisit:         PlanVisitSheet()
            case .about:             AboutSheet()
            case .notes(let s):      SermonNotesSheet(sermon: s)
            }
        }
    }
}
```

External links go through an explicit allow-list. Only the organization's own site and YouTube render in-app. Everything else, including giving pages, opens in the default browser. Hosts are suffix-matched, so a lookalike domain fails closed:

```swift
enum LinkPolicy {
    private static let inAppHosts = ["example-church.org", "youtube.com", "youtu.be"]

    static func opensInApp(_ url: URL) -> Bool {
        guard let host = url.host?.lowercased() else { return false }
        return inAppHosts.contains { host == $0 || host.hasSuffix(".\($0)") }
    }
}
```

## 6. Photo cards that don't break layout (or taps)

`Image().resizable().scaledToFill()` inside a height-constrained stack reports an oversized ideal width and pushes the whole screen past its edge. `.clipped()` hides the overflow visually, but the overflowing image is still hit-testable, so on wide layouts like iPad it covers neighboring buttons. Every photo tile goes through one wrapper:

```swift
struct FillImage: View {
    let name: String
    var body: some View {
        Color.clear                                          // takes exactly the proposed size
            .overlay(Image(name).resizable().scaledToFill()) // image can't inflate layout
            .clipped()                                       // clips drawing...
            .contentShape(Rectangle())                       // ...and the touch region
    }
}
```

## 7. Local-first features

Prayer walls, sermon notes, saved messages, booking requests, journals and pre-op checklists all persist to `UserDefaults` through small `ObservableObject` stores that encode `Codable` models. No account or server is needed to use the app, and nothing personal leaves the device. Booking-style apps compose a prefilled email (`MFMailComposeViewController`, with a `mailto:` fallback) rather than taking payments in-app.

Other platform features used across the apps:

- `UNUserNotificationCenter` local notifications: weekly service reminder, daily verse, RSVP nudges.
- `ImageRenderer` + `ShareLink` for 1080x1920 shareable verse cards.
- `AVPlayerLooper` full-bleed landing films and muted background loops.
- A minimal ICS parser that turns an organization's public Google Calendar feeds into a native events list.
- `matchedGeometryEffect` custom tab bars, `scrollTransition` reveals and `sensoryFeedback` haptics.

## 8. Screenshot automation

Most of the apps accept a debug launch argument that opens a given tab and skips the splash or landing film, so store screenshots are scripted with `simctl`. Launch arguments arrive as strings, so the value is read through `UserDefaults` rather than cast:

```swift
.onAppear {
    if UserDefaults.standard.object(forKey: "selectedTab") != nil {   // passed as -selectedTab 2
        selection = UserDefaults.standard.integer(forKey: "selectedTab")
        showLanding = false
    }
}
```

```sh
xcrun simctl launch booted com.example.app -selectedTab 2
xcrun simctl io booted screenshot 03-watch.png
```

## 9. Beyond the church shell

- **Village Applause** adds a host-authoritative live quiz engine (Kahoot-style). The teacher's phone owns game state and scoring and reveals answers only at reveal time. Students join over Bonjour peer-to-peer or through a small Node WebSocket relay, and both sit behind the same `LiveHostTransport` / `LivePlayerTransport` protocols, alongside a demo transport that simulates a classroom on one phone.
- **The Men's Winning Circle** is Expo + Supabase: a daily five-dimension check-in, momentum scoring with grace decay instead of hard streak resets, and small accountability pods.
