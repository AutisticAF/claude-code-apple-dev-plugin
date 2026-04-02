---
name: gamekit
description: GameKit patterns for Game Center authentication, leaderboards, achievements, matchmaking, and challenges. Use when integrating Game Center features.
---

> **First step:** Tell the user: "gamekit skill loaded."

# GameKit and Game Center

## When This Skill Activates

Use this skill when the user:
- Wants to integrate Game Center into their app or game
- Asks about player authentication with GKLocalPlayer
- Needs leaderboard or achievement setup and submission
- Wants real-time or turn-based multiplayer matchmaking
- Asks about GKMatch, GKTurnBasedMatch, or GKMatchmakerViewController
- Needs to display the Game Center dashboard or access point
- Wants to issue or handle player-to-player challenges
- Asks about App Store Connect configuration for leaderboards or achievements
- Needs SwiftUI integration for Game Center view controllers

## Decision Tree: GameKit Features

| Goal | Key Classes | Notes |
|------|-------------|-------|
| Authenticate player | **GKLocalPlayer** | Required before any other GameKit call |
| Show Game Center dashboard | **GKAccessPoint**, **GKGameCenterViewController** | Overlay or full-screen |
| Submit and display scores | **GKLeaderboard**, **GKLeaderboardScore** | Configure in App Store Connect first |
| Track player progress | **GKAchievement**, **GKAchievementDescription** | Percentage-based (0-100) |
| Real-time multiplayer | **GKMatchRequest**, **GKMatchmakerViewController**, **GKMatch** | Peer-to-peer data exchange |
| Turn-based multiplayer | **GKTurnBasedMatchmakerViewController**, **GKTurnBasedMatch** | Asynchronous turns with match data |
| Player challenges | **GKChallenge** | Score or achievement challenges |

## API Availability

| API | iOS | macOS | tvOS | Notes |
|-----|-----|-------|------|-------|
| GKLocalPlayer.authenticateHandler | 6.0+ | 10.9+ | 9.0+ | Required entry point |
| GKAccessPoint | 14.0+ | 11.0+ | 14.0+ | Floating dashboard button |
| GKLeaderboard (modern) | 14.0+ | 11.0+ | 14.0+ | Replaces legacy class methods |
| GKAchievement | 4.1+ | 10.8+ | 9.0+ | Report with percentComplete |
| GKMatchmakerViewController | 4.1+ | 10.8+ | 9.0+ | Real-time match UI |
| GKTurnBasedMatchmakerViewController | 5.0+ | 10.8+ | 9.0+ | Turn-based match UI |
| GKGameCenterViewController | 6.0+ | 10.9+ | 9.0+ | Full dashboard screen |

## Authentication

Set the handler early in your app lifecycle (e.g., `application(_:didFinishLaunchingWithOptions:)` or App `init`). Required before any other GameKit call.

```swift
import GameKit

func authenticatePlayer() {
    GKLocalPlayer.local.authenticateHandler = { viewController, error in
        if let vc = viewController {
            // Present the Game Center sign-in view controller
            // In UIKit: present(vc, animated: true)
            return
        }

        if let error = error {
            print("Game Center auth failed: \(error.localizedDescription)")
            return
        }

        if GKLocalPlayer.local.isAuthenticated {
            print("Authenticated as \(GKLocalPlayer.local.displayName)")
            // Enable Game Center features
            GKAccessPoint.shared.isActive = true
        }
    }
}
```

## Access Point

GKAccessPoint provides a floating Game Center button that opens the dashboard overlay.

```swift
import GameKit

func configureAccessPoint() {
    GKAccessPoint.shared.location = .topLeading
    GKAccessPoint.shared.showHighlights = true
    GKAccessPoint.shared.isActive = true
}

// Programmatically trigger the dashboard to a specific state
func showLeaderboards() {
    GKAccessPoint.shared.trigger(state: .leaderboards) {}
}
```

## Leaderboards

### App Store Connect Setup

In App Store Connect > Services > Game Center, add a Leaderboard (classic or recurring) with a unique ID (e.g., `com.yourcompany.game.highscore`). Configure sort order, score format, and localizations. For leaderboard sets, create a GKLeaderboardSet and assign leaderboards to it.

### Submitting Scores

```swift
import GameKit

func submitScore(_ score: Int, leaderboardID: String) async throws {
    guard GKLocalPlayer.local.isAuthenticated else { return }

    try await GKLeaderboard.submitScore(
        score,
        context: 0,
        player: GKLocalPlayer.local,
        leaderboardIDs: [leaderboardID]
    )
}
```

### Loading Leaderboard Entries

```swift
func loadTopScores(leaderboardID: String) async throws {
    let leaderboards = try await GKLeaderboard.loadLeaderboards(IDs: [leaderboardID])
    guard let lb = leaderboards.first else { return }
    let (localEntry, entries, _) = try await lb.loadEntries(
        for: .global, timeScope: .allTime, range: NSRange(location: 1, length: 10)
    )
    // entries contains top 10; localEntry is the current player's entry (may be nil)
}
```

## Achievements

### App Store Connect Setup

In App Store Connect > Services > Game Center > Achievements, add an Achievement with a unique ID. Set point value (total across all achievements must not exceed 1000). Configure as hidden or visible and add localized title, description (earned and unearned), and image.

### Reporting Progress

```swift
import GameKit

func reportAchievement(id: String, percentComplete: Double) async throws {
    guard GKLocalPlayer.local.isAuthenticated else { return }
    let achievement = GKAchievement(identifier: id)
    achievement.percentComplete = percentComplete
    achievement.showsCompletionBanner = true
    try await GKAchievement.report([achievement])
}
```

Report multiple achievements by passing an array to `GKAchievement.report(_:)`.

### Loading Achievement Descriptions

```swift
func loadAchievements() async throws {
    let descriptions = try await GKAchievementDescription.loadAchievementDescriptions()
    let progress = try await GKAchievement.loadAchievements()
    for desc in descriptions {
        let percent = progress.first { $0.identifier == desc.identifier }?.percentComplete ?? 0
        print("\(desc.title): \(percent)% complete")
    }
}
```

## Real-Time Matchmaking

### Presenting the Matchmaker and Handling the Match

```swift
import GameKit
import UIKit

class GameViewController: UIViewController, GKMatchmakerViewControllerDelegate, GKMatchDelegate {
    var currentMatch: GKMatch?

    func findMatch() {
        let request = GKMatchRequest()
        request.minPlayers = 2
        request.maxPlayers = 4
        guard let vc = GKMatchmakerViewController(matchRequest: request) else { return }
        vc.matchmakerDelegate = self
        present(vc, animated: true)
    }

    // GKMatchmakerViewControllerDelegate
    func matchmakerViewController(_ vc: GKMatchmakerViewController, didFind match: GKMatch) {
        vc.dismiss(animated: true)
        currentMatch = match
        match.delegate = self
    }
    func matchmakerViewControllerWasCancelled(_ vc: GKMatchmakerViewController) {
        vc.dismiss(animated: true)
    }
    func matchmakerViewController(_ vc: GKMatchmakerViewController, didFailWithError error: Error) {
        vc.dismiss(animated: true)
    }

    // GKMatchDelegate - receiving data and connection changes
    func match(_ match: GKMatch, didReceive data: Data, fromRemotePlayer player: GKPlayer) {
        // Decode and handle incoming game data
    }
    func match(_ match: GKMatch, player: GKPlayer, didChange state: GKPlayerConnectionState) {
        // Handle .connected / .disconnected
    }

    // Sending data to other players
    func sendGameData(_ data: Data) {
        try? currentMatch?.sendData(toAllPlayers: data, with: .reliable)
    }
}
```

### Data Delivery Modes

| Mode | Use Case |
|------|----------|
| `.reliable` | Critical game state: scores, turn actions, checkpoints |
| `.unreliable` | Frequent updates: real-time position, animation (tolerate loss) |

## Turn-Based Matches

```swift
import GameKit

// Present the turn-based matchmaker
func findTurnBasedMatch(from presenter: UIViewController) {
    let request = GKMatchRequest()
    request.minPlayers = 2
    request.maxPlayers = 4
    guard let vc = GKTurnBasedMatchmakerViewController(matchRequest: request) else { return }
    vc.turnBasedMatchmakerDelegate = self // Implement GKTurnBasedMatchmakerViewControllerDelegate
    presenter.present(vc, animated: true)
}

// Take a turn: update match data and pass to next participant
func takeTurn(match: GKTurnBasedMatch, gameData: Data, nextPlayer: GKPlayer) async throws {
    let nextParticipants = match.participants.filter {
        $0.player?.gamePlayerID == nextPlayer.gamePlayerID
    }
    try await match.endTurn(
        withNextParticipants: nextParticipants,
        turnTimeout: GKTurnTimeoutDefault,
        match: gameData
    )
}

// End the match: set outcomes for all participants
func endMatch(_ match: GKTurnBasedMatch, finalData: Data) async throws {
    for participant in match.participants {
        participant.matchOutcome = participant.player == GKLocalPlayer.local ? .won : .lost
    }
    try await match.endMatchInTurn(withMatch: finalData)
}
```

Always dismiss the turn-based matchmaker in both `turnBasedMatchmakerViewControllerWasCancelled(_:)` and `turnBasedMatchmakerViewController(_:didFailWithError:)`.

## Challenges

Players can challenge others to beat a score or earn an achievement via `GKChallenge`.

```swift
import GameKit

// Issue a score challenge from a leaderboard entry
func issueScoreChallenge(score: GKLeaderboard.Entry, to players: [GKPlayer]) async throws {
    let controller = try await score.challengeComposeController(
        withMessage: "Beat my score!", players: players
    )
    // Present the returned view controller
}

// Load challenges received from other players
let challenges = try await GKChallenge.loadReceivedChallenges()
```

## SwiftUI Integration

Wrap GKGameCenterViewController using UIViewControllerRepresentable:

```swift
import SwiftUI
import GameKit

struct GameCenterView: UIViewControllerRepresentable {
    let viewState: GKGameCenterViewController.State

    func makeUIViewController(context: Context) -> GKGameCenterViewController {
        let gc = GKGameCenterViewController(state: viewState)
        gc.gameCenterDelegate = context.coordinator
        return gc
    }
    func updateUIViewController(_ vc: GKGameCenterViewController, context: Context) {}
    func makeCoordinator() -> Coordinator { Coordinator() }

    class Coordinator: NSObject, GKGameCenterControllerDelegate {
        func gameCenterViewControllerDidFinish(_ c: GKGameCenterViewController) {
            c.dismiss(animated: true)
        }
    }
}

// Usage: .sheet(isPresented: $show) { GameCenterView(viewState: .leaderboards) }
```

Available states: `.default`, `.leaderboards`, `.achievements`, `.localPlayerProfile`, `.dashboard`, `.localPlayerFriendsList`.

## App Store Connect Setup Summary

1. **Enable Game Center** capability in Xcode and App Store Connect
2. **Leaderboards**: Services > Game Center > Leaderboards -- set unique IDs, score format, sort order
3. **Achievements**: Services > Game Center > Achievements -- unique IDs, point values (max 1000 total)
4. **Multiplayer**: No additional App Store Connect config needed for matchmaking
5. **Testing**: Use sandbox accounts; Game Center works in TestFlight builds

## Common Pitfalls and Patterns

```swift
// ✅ Good: Set authenticateHandler early, check isAuthenticated before API calls
GKLocalPlayer.local.authenticateHandler = { vc, error in /* ... */ }

// ❌ Bad: Calling GameKit APIs before authentication completes
GKLeaderboard.submitScore(100, context: 0, player: GKLocalPlayer.local,
                          leaderboardIDs: ["id"]) // May silently fail

// ✅ Good: Use the modern async class method for scores
try await GKLeaderboard.submitScore(500, context: 0,
    player: GKLocalPlayer.local, leaderboardIDs: ["com.game.highscore"])

// ❌ Bad: Using deprecated GKScore API
let score = GKScore(leaderboardIdentifier: "com.game.highscore") // Deprecated

// ✅ Good: Report total cumulative percentage for achievements
achievement.percentComplete = min(currentKills / requiredKills * 100.0, 100.0)

// ❌ Bad: Reporting incremental progress (GameKit keeps highest value, not sum)
achievement.percentComplete = 10.0 // Does not add to previous value

// ✅ Good: Always dismiss matchmaker VC in every delegate callback
func matchmakerViewControllerWasCancelled(_ vc: GKMatchmakerViewController) {
    vc.dismiss(animated: true)
}
// ❌ Bad: Forgetting to dismiss on cancel or error (leaves UI stuck)

// ✅ Good: Use .reliable for critical state, .unreliable for frequent updates
try match.sendData(toAllPlayers: stateData, with: .reliable)
try match.sendData(toAllPlayers: positionData, with: .unreliable)

// ❌ Bad: Sending frequent position updates with .reliable (causes latency spikes)
try match.sendData(toAllPlayers: positionData, with: .reliable) // 60 times/sec = bad
```
