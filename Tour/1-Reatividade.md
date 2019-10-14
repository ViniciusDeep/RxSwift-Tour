# Programação Reativa

Programação reativa é um paradgma de programação orientado ao fluxo de dados assíncrono, que está ligado diretamente ao registro e acontecimento de eventos.

Existem várias formas e conceitos necessários quando se estuda programação reativa, um deles é um padrão de projeto chamado Observer que consiste em um cojunto de componentes que são necessários para montar uma seguinte logistica

O Observer é um padrão de projeto de software que define uma dependência um-para-muitos entre objetos de modo que quando um objeto muda o estado, todos seus dependentes são notificados e atualizados automaticamente. Permite que objetos interessados sejam avisados da mudança de estado ou outros eventos ocorrendo num outro objeto.

O padrão Observer é também chamado de Publisher-Subscriber, Event Generator e Dependents.


Exemplo de implementação de um Observer em Swift, para caso de curiosidade. 

```swift 
class AudioPlayer {
    private var state = State.idle {
        // We add a property observer on 'state', which lets us
        // run a function on each value change.
        didSet { stateDidChange() }
    }

    func play(_ item: Item) {
        state = .playing(item)
        startPlayback(with: item)
    }

    func pause() {
        switch state {
        case .idle, .paused:
            // Calling pause when we're not in a playing state
            // could be considered a programming error, but since
            // it doesn't do any harm, we simply break here.
            break
        case .playing(let item):
            state = .paused(item)
            pausePlayback()
        }
    }

    func stop() {
        state = .idle
        stopPlayback()
    }
}
```

```swift
private extension AudioPlayer {
    enum State {
        case idle
        case playing(Item)
        case paused(Item)
    }
}
```

````swift
class AudioPlayer {
    private let notificationCenter: NotificationCenter

    init(notificationCenter: NotificationCenter = .default) {
        self.notificationCenter = notificationCenter
    }
}
```

```swift
extension Notification.Name {
    static var playbackStarted: Notification.Name {
        return .init(rawValue: "AudioPlayer.playbackStarted")
    }

    static var playbackPaused: Notification.Name {
        return .init(rawValue: "AudioPlayer.playbackPaused")
    }

    static var playbackStopped: Notification.Name {
        return .init(rawValue: "AudioPlayer.playbackStopped")
    }
}
```

```swift
private extension AudioPlayer {
    func stateDidChange() {
        switch state {
        case .idle:
            notificationCenter.post(name: .playbackStopped, object: nil)
        case .playing(let item):
            notificationCenter.post(name: .playbackStarted, object: item)
        case .paused(let item):
            notificationCenter.post(name: .playbackPaused, object: item)
        }
    }
}
```
```swift
class NowPlayingViewController: UIViewController {
    deinit {
        // If your app supports iOS 8 or earlier, you need to manually
        // remove the observer from the center. In later versions
        // this is done automatically.
        notificationCenter.removeObserver(self)
    }

    override func viewDidLoad() {
        super.viewDidLoad()

        notificationCenter.addObserver(self,
            selector: #selector(playbackDidStart),
            name: .playbackStarted,
            object: nil
        )
    }

    @objc private func playbackDidStart(_ notification: Notification) {
        guard let item = notification.object as? AudioPlayer.Item else {
            let object = notification.object as Any
            assertionFailure("Invalid object: \(object)")
            return
        }

        titleLabel.text = item.title
        durationLabel.text = "\(item.duration)"
    }
}
```

```swift
protocol AudioPlayerObserver: class {
    func audioPlayer(_ player: AudioPlayer,
                     didStartPlaying item: AudioPlayer.Item)

    func audioPlayer(_ player: AudioPlayer, 
                     didPausePlaybackOf item: AudioPlayer.Item)

    func audioPlayerDidStop(_ player: AudioPlayer)
}
```

```swift
extension AudioPlayerObserver {
    func audioPlayer(_ player: AudioPlayer,
                     didStartPlaying item: AudioPlayer.Item) {}

    func audioPlayer(_ player: AudioPlayer, 
                     didPausePlaybackOf item: AudioPlayer.Item) {}

    func audioPlayerDidStop(_ player: AudioPlayer) {}
}
```

```swift
private extension AudioPlayer {
    struct Observation {
        weak var observer: AudioPlayerObserver?
    }
}
```

```swift
class AudioPlayer {
    private var observations = [ObjectIdentifier : Observation]()
}
```

```swift 
private extension AudioPlayer {
    func stateDidChange() {
        for (id, observation) in observations {
            // If the observer is no longer in memory, we
            // can clean up the observation for its ID
            guard let observer = observation.observer else {
                observations.removeValue(forKey: id)
                continue
            }

            switch state {
            case .idle:
                observer.audioPlayerDidStop(self)
            case .playing(let item):
                observer.audioPlayer(self, didStartPlaying: item)
            case .paused(let item):
                observer.audioPlayer(self, didPausePlaybackOf: item)
            }
        }
    }
}
```

```swift
extension AudioPlayer {
    func addObserver(_ observer: AudioPlayerObserver) {
        let id = ObjectIdentifier(observer)
        observations[id] = Observation(observer: observer)
    }

    func removeObserver(_ observer: AudioPlayerObserver) {
        let id = ObjectIdentifier(observer)
        observations.removeValue(forKey: id)
    }
}
```

```swift
class NowPlayingViewController: UIViewController {
    override func viewDidLoad() {
        super.viewDidLoad()
        player.addObserver(self)
    }
}
```

```swift
extension NowPlayingViewController: AudioPlayerObserver {
    func audioPlayer(_ player: AudioPlayer,
                     didStartPlaying item: AudioPlayer.Item) {
        titleLabel.text = item.title
        durationLabel.text = "\(item.duration)"
    }
}
```







