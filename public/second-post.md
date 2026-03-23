---
title: 'SwiftUI + Firebase（Auth & Firestore）を導入して認証とDB機能を実装する'
tags:
  - name: 'Swift'
  - name: 'SwiftUI'
  - name: 'Firebase'
  - name: 'iOS'
  - name: 'Firestore'
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
---

## はじめに

SwiftUIで開発中の言語学習アプリ「Mochibo」に、Firebase Authentication（メール/パスワード認証）と Cloud Firestore（データベース）を導入しました。

もともとSwiftDataでローカル保存していたフレーズデータを、Firestoreに移行してユーザーごとにクラウド同期する構成にしています。

この記事では、セットアップから実装まで一通りまとめます。

## 環境

- Xcode 16
- iOS 18+
- Swift 5.9+
- Firebase iOS SDK（SPM）

## 全体構成

```
Mochibo/
├── MochiboApp.swift              # Firebase初期化
├── ContentView.swift             # 認証画面 ↔ メイン画面の切り替え
├── Services/
│   ├── AuthService.swift         # Firebase Auth ラッパー
│   └── FirestoreService.swift    # Firestore CRUD + リアルタイムリスナー
├── Models/
│   └── Phrase.swift              # Codable構造体（旧SwiftData @Model）
└── Views/
    ├── MainTabView.swift
    ├── MyPhrasesView.swift       # フレーズ一覧（Firestore連携）
    ├── ProfileView.swift         # ログアウト機能
    └── Phrases/
        ├── AddPhraseView.swift   # フレーズ追加（Firestore書き込み）
        └── PracticeView.swift
```

**Firestoreのデータ構造：**

```
users/
  └── {uid}/
      └── phrases/
          └── {phraseId}/
              ├── originalText: String
              ├── translatedText: String
              └── createdAt: Timestamp
```

## 1. Firebaseセットアップ

### Firebase Console側

1. [Firebase Console](https://console.firebase.google.com) でプロジェクト作成
2. iOSアプリを追加（Bundle IDを入力）
3. `GoogleService-Info.plist` をダウンロード → Xcodeプロジェクトに配置
4. **Authentication** → Sign-in method → **メール/パスワード** を有効化
5. **Firestore Database** → データベースを作成 → テストモードで開始

### Xcode側（SPM）

**File → Add Package Dependencies** から以下を追加：

```
https://github.com/firebase/firebase-ios-sdk
```

追加するライブラリ：
- `FirebaseAuth`
- `FirebaseFirestore`

## 2. Firebase初期化（MochiboApp.swift）

`AppDelegate`で`FirebaseApp.configure()`を呼び、`AuthService`と`FirestoreService`を`@StateObject`で保持します。

```swift
import SwiftUI
import FirebaseCore

class AppDelegate: NSObject, UIApplicationDelegate {
    func application(
        _ application: UIApplication,
        didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]? = nil
    ) -> Bool {
        FirebaseApp.configure()
        return true
    }
}

@main
struct MochiboApp: App {
    @UIApplicationDelegateAdaptor(AppDelegate.self) var delegate

    @StateObject private var authService = AuthService()
    @StateObject private var firestoreService = FirestoreService()

    var body: some Scene {
        WindowGroup {
            ContentView()
                .environmentObject(authService)
                .environmentObject(firestoreService)
                .onAppear {
                    authService.onSignOut = { [weak firestoreService] in
                        firestoreService?.stopListening()
                    }
                }
        }
    }
}
```

**ポイント：** サインアウト時に`stopListening()`を呼んでFirestoreリスナーを解除しています。`onDisappear`ではなく認証状態の変化に連動させることで、アプリのバックグラウンド遷移などで意図せずリスナーが切れるのを防ぎます。

## 3. AuthService（認証サービス）

Firebase Authをラップした`ObservableObject`です。`addStateDidChangeListener`で認証状態をリアルタイムに監視します。

```swift
import Foundation
import FirebaseAuth

@MainActor
final class AuthService: ObservableObject {
    @Published var currentUser: User?
    @Published var isAuthenticated = false
    @Published var errorMessage: String?
    @Published var isLoading = false

    var onSignOut: (() -> Void)?

    private var handle: AuthStateDidChangeListenerHandle?

    init() {
        handle = Auth.auth().addStateDidChangeListener { [weak self] _, user in
            Task { @MainActor in
                let wasAuthenticated = self?.isAuthenticated ?? false
                self?.currentUser = user
                self?.isAuthenticated = user != nil
                if wasAuthenticated && user == nil {
                    self?.onSignOut?()
                }
            }
        }
    }

    deinit {
        if let handle { Auth.auth().removeStateDidChangeListener(handle) }
    }

    func signUp(email: String, password: String) async {
        isLoading = true
        errorMessage = nil
        do {
            try await Auth.auth().createUser(withEmail: email, password: password)
        } catch {
            errorMessage = error.localizedDescription
        }
        isLoading = false
    }

    func signIn(email: String, password: String) async {
        isLoading = true
        errorMessage = nil
        do {
            try await Auth.auth().signIn(withEmail: email, password: password)
        } catch {
            errorMessage = error.localizedDescription
        }
        isLoading = false
    }

    func resetPassword(email: String) async -> Bool {
        isLoading = true
        errorMessage = nil
        do {
            try await Auth.auth().sendPasswordReset(withEmail: email)
            isLoading = false
            return true
        } catch {
            errorMessage = error.localizedDescription
            isLoading = false
            return false
        }
    }

    func signOut() {
        do {
            try Auth.auth().signOut()
        } catch {
            errorMessage = error.localizedDescription
        }
    }
}
```

**設計のポイント：**
- `isLoading`で各操作中のローディング状態を管理
- `errorMessage`でUIにエラーを表示可能
- `onSignOut`コールバックで外部（Firestoreリスナー解除など）に通知

## 4. FirestoreService（データベースサービス）

ユーザーごとの`phrases`サブコレクションに対してCRUD操作とリアルタイム同期を行います。

```swift
import Foundation
import FirebaseAuth
import FirebaseFirestore

@MainActor
final class FirestoreService: ObservableObject {
    @Published var phrases: [Phrase] = []
    @Published var isLoading = false

    private let db = Firestore.firestore()
    private var listener: ListenerRegistration?

    private var phrasesCollection: CollectionReference? {
        guard let uid = Auth.auth().currentUser?.uid else { return nil }
        return db.collection("users").document(uid).collection("phrases")
    }

    func startListening() {
        guard let collection = phrasesCollection else { return }
        isLoading = true

        listener = collection
            .order(by: "createdAt", descending: true)
            .addSnapshotListener { [weak self] snapshot, error in
                Task { @MainActor in
                    guard let self else { return }
                    self.isLoading = false

                    if let error {
                        print("Firestore listen error: \(error.localizedDescription)")
                        return
                    }

                    self.phrases = snapshot?.documents.compactMap { doc in
                        try? doc.data(as: Phrase.self)
                    } ?? []
                }
            }
    }

    func stopListening() {
        listener?.remove()
        listener = nil
        phrases = []
    }

    func addPhrase(originalText: String, translatedText: String) async throws {
        guard let collection = phrasesCollection else { return }
        let phrase = Phrase(originalText: originalText, translatedText: translatedText)
        try collection.addDocument(from: phrase)
    }

    func deletePhrase(_ phrase: Phrase) async throws {
        guard let collection = phrasesCollection,
              let id = phrase.id else { return }
        try await collection.document(id).delete()
    }
}
```

**ポイント：** `addSnapshotListener`を使うことで、追加・削除の結果がリアルタイムにUIへ反映されます。SwiftDataの`@Query`と同じような感覚で使えます。

## 5. Phraseモデル（SwiftData → Firestore）

SwiftDataの`@Model`クラスから、Firestoreの`Codable`構造体に変更しました。

**Before（SwiftData）：**
```swift
import SwiftData

@Model
final class Phrase {
    var originalText: String
    var translatedText: String
    var createdAt: Date
}
```

**After（Firestore）：**
```swift
import Foundation
import FirebaseFirestore

struct Phrase: Codable, Identifiable {
    @DocumentID var id: String?
    var originalText: String
    var translatedText: String
    var createdAt: Date

    init(originalText: String, translatedText: String, createdAt: Date = .now) {
        self.originalText = originalText
        self.translatedText = translatedText
        self.createdAt = createdAt
    }
}
```

`@DocumentID`を付けることで、FirestoreのドキュメントIDが自動的にマッピングされます。

## 6. 認証画面の切り替え（ContentView）

`authService.isAuthenticated`で認証画面とメイン画面を切り替えます。

```swift
struct ContentView: View {
    @EnvironmentObject var authService: AuthService
    @EnvironmentObject var firestoreService: FirestoreService

    var body: some View {
        if authService.isAuthenticated {
            MainTabView(language: $language)
                .onAppear { firestoreService.startListening() }
        } else {
            authView
        }
    }
}
```

### エラー表示

Firebase Authのエラーメッセージをバナーで表示します。

```swift
@ViewBuilder
private var errorBanner: some View {
    if let error = authService.errorMessage {
        HStack(spacing: 8) {
            Image(systemName: "exclamationmark.triangle.fill")
                .font(.system(size: 14))
            Text(error)
                .font(.system(size: 12, design: .rounded))
                .lineLimit(3)
        }
        .foregroundColor(.red)
        .padding(12)
        .frame(maxWidth: .infinity, alignment: .leading)
        .background(
            RoundedRectangle(cornerRadius: 12)
                .fill(.red.opacity(0.08))
        )
        .transition(.opacity.combined(with: .offset(y: 8)))
    }
}
```

### サインアップ時のバリデーション

```swift
MochiButton(title: l.signUp) {
    guard password == confirmPassword else {
        authService.errorMessage = l.passwordMismatch
        return
    }
    Task { await authService.signUp(email: email, password: password) }
}
```

## 7. View側のSwiftData → Firestore移行

### MyPhrasesView

**Before：**
```swift
@Query(sort: \Phrase.createdAt, order: .reverse) private var phrases: [Phrase]
@Environment(\.modelContext) private var modelContext

// 削除
modelContext.delete(phrase)
```

**After：**
```swift
@EnvironmentObject var firestoreService: FirestoreService

// 参照
firestoreService.phrases

// 削除
Task { try? await firestoreService.deletePhrase(phrase) }
```

### AddPhraseView

**Before：**
```swift
let phrase = Phrase(originalText: ..., translatedText: ...)
modelContext.insert(phrase)
```

**After：**
```swift
try await firestoreService.addPhrase(originalText: ..., translatedText: ...)
```

## 8. ログアウト機能（ProfileView）

```swift
struct ProfileView: View {
    @EnvironmentObject var authService: AuthService

    var body: some View {
        // ...
        Button {
            authService.signOut()
        } label: {
            HStack(spacing: 8) {
                Image(systemName: "rectangle.portrait.and.arrow.right")
                Text("ログアウト")
            }
            .foregroundColor(.red)
            .frame(maxWidth: .infinity)
            .padding(.vertical, 12)
            .background(
                RoundedRectangle(cornerRadius: 16)
                    .fill(.red.opacity(0.08))
            )
        }
    }
}
```

## まとめ

| 項目 | Before | After |
|------|--------|-------|
| 認証 | なし（ボタンで即遷移） | Firebase Auth |
| DB | SwiftData（ローカル） | Cloud Firestore |
| モデル | `@Model` class | `Codable` struct + `@DocumentID` |
| データ取得 | `@Query` | `addSnapshotListener`（リアルタイム） |
| データ書き込み | `modelContext.insert/delete` | `addDocument` / `document.delete` |
| ユーザー分離 | なし | `users/{uid}/phrases/` |

SwiftDataからFirestoreへの移行は、モデルを`Codable`構造体に変え、`@Query`の代わりにリアルタイムリスナーを使う形になります。`EnvironmentObject`を使えば、SwiftDataの`@Environment(\.modelContext)`と同じ感覚で各Viewにサービスを注入できます。
