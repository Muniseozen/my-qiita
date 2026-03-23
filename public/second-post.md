---
title: 'SwiftUI + Firebaseで認証とDB機能を実装する'
tags:
  - name: 'Swift'
  - name: 'SwiftUI'
  - name: 'Firebase'
  - name: 'iOS'
  - name: 'Firestore'
private: false
updated_at: null
id: null
organization_url_name: null
slide: false
---

## はじめに

SwiftUIアプリにFirebase Authentication（メール/パスワード認証）と Cloud Firestore（データベース）を導入する手順をまとめます。

この記事では、簡単なメモアプリを例に、**ユーザー登録・ログイン・データのCRUD・リアルタイム同期**までを一通り実装します。

## 環境

- Xcode 16
- iOS 18+
- Swift 5.9+
- Firebase iOS SDK（SPM）

## 完成イメージ

```
MyApp/
├── MyApp.swift                # Firebase初期化
├── ContentView.swift          # 認証 ↔ メイン画面の切り替え
├── Services/
│   ├── AuthService.swift      # Firebase Auth ラッパー
│   └── FirestoreService.swift # Firestore CRUD + リアルタイムリスナー
├── Models/
│   └── Memo.swift             # Firestoreモデル
└── Views/
    ├── HomeView.swift         # メモ一覧
    ├── AddMemoView.swift      # メモ追加
    └── ProfileView.swift      # ログアウト
```

**Firestoreのデータ構造：**

```
users/
  └── {uid}/
      └── memos/
          └── {memoId}/
              ├── title: String
              ├── body: String
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

![SPMでFirebase SDKを追加](https://raw.githubusercontent.com/Muniseozen/my-qiita/main/public/images/add-package.png)

パッケージ製品の選択画面で以下の2つにチェック：
- `FirebaseAuth`
- `FirebaseFirestore`

## 2. Firebase初期化（App.swift）

`AppDelegate`で`FirebaseApp.configure()`を呼び、サービスを`@StateObject`で保持して`environmentObject`で配布します。

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
struct MyApp: App {
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

**ポイント：** サインアウト時に`stopListening()`でFirestoreリスナーを解除します。`onDisappear`ではなく認証状態の変化に連動させることで、バックグラウンド遷移などで意図せずリスナーが切れるのを防ぎます。

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

    // MARK: - サインアップ

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

    // MARK: - サインイン

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

    // MARK: - パスワードリセット

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

    // MARK: - サインアウト

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
- `isLoading` → ボタンの無効化やProgressView表示に使用
- `errorMessage` → UIにエラーバナーを表示
- `onSignOut` → Firestoreリスナー解除など外部への通知用

## 4. FirestoreService（データベースサービス）

ユーザーごとのサブコレクションに対してCRUD操作とリアルタイム同期を行います。

```swift
import Foundation
import FirebaseAuth
import FirebaseFirestore

@MainActor
final class FirestoreService: ObservableObject {
    @Published var memos: [Memo] = []
    @Published var isLoading = false

    private let db = Firestore.firestore()
    private var listener: ListenerRegistration?

    // MARK: - コレクション参照

    private var memosCollection: CollectionReference? {
        guard let uid = Auth.auth().currentUser?.uid else { return nil }
        return db.collection("users").document(uid).collection("memos")
    }

    // MARK: - リアルタイムリスナー

    func startListening() {
        guard let collection = memosCollection else { return }
        isLoading = true

        listener = collection
            .order(by: "createdAt", descending: true)
            .addSnapshotListener { [weak self] snapshot, error in
                Task { @MainActor in
                    guard let self else { return }
                    self.isLoading = false

                    if let error {
                        print("Firestore error: \(error.localizedDescription)")
                        return
                    }

                    self.memos = snapshot?.documents.compactMap { doc in
                        try? doc.data(as: Memo.self)
                    } ?? []
                }
            }
    }

    func stopListening() {
        listener?.remove()
        listener = nil
        memos = []
    }

    // MARK: - 追加

    func addMemo(title: String, body: String) async throws {
        guard let collection = memosCollection else { return }
        let memo = Memo(title: title, body: body)
        try collection.addDocument(from: memo)
    }

    // MARK: - 削除

    func deleteMemo(_ memo: Memo) async throws {
        guard let collection = memosCollection,
              let id = memo.id else { return }
        try await collection.document(id).delete()
    }
}
```

`addSnapshotListener`を使うことで、データの追加・削除がリアルタイムにUIへ反映されます。

## 5. Firestoreモデル

`@DocumentID`を付けると、FirestoreのドキュメントIDが自動的にマッピングされます。

```swift
import Foundation
import FirebaseFirestore

struct Memo: Codable, Identifiable {
    @DocumentID var id: String?
    var title: String
    var body: String
    var createdAt: Date

    init(title: String, body: String, createdAt: Date = .now) {
        self.title = title
        self.body = body
        self.createdAt = createdAt
    }
}
```

## 6. 認証画面の切り替え

`authService.isAuthenticated`の値でビューを切り替えます。認証成功後に`startListening()`でFirestoreの同期を開始します。

```swift
struct ContentView: View {
    @EnvironmentObject var authService: AuthService
    @EnvironmentObject var firestoreService: FirestoreService

    @State private var email = ""
    @State private var password = ""
    @State private var isSignUp = true

    var body: some View {
        if authService.isAuthenticated {
            HomeView()
                .onAppear { firestoreService.startListening() }
        } else {
            loginView
        }
    }

    private var loginView: some View {
        VStack(spacing: 16) {
            TextField("メールアドレス", text: $email)
                .textFieldStyle(.roundedBorder)
                .keyboardType(.emailAddress)
                .autocapitalization(.none)

            SecureField("パスワード", text: $password)
                .textFieldStyle(.roundedBorder)

            // エラー表示
            if let error = authService.errorMessage {
                Text(error)
                    .foregroundColor(.red)
                    .font(.caption)
            }

            Button(isSignUp ? "新規登録" : "ログイン") {
                Task {
                    if isSignUp {
                        await authService.signUp(email: email, password: password)
                    } else {
                        await authService.signIn(email: email, password: password)
                    }
                }
            }
            .disabled(authService.isLoading)

            if authService.isLoading {
                ProgressView()
            }

            Button(isSignUp ? "ログインはこちら" : "新規登録はこちら") {
                isSignUp.toggle()
                authService.errorMessage = nil
            }
            .font(.caption)
        }
        .padding(24)
    }
}
```

## 7. メモ一覧画面

`firestoreService.memos`を参照するだけで、リアルタイムにデータが反映されます。

```swift
struct HomeView: View {
    @EnvironmentObject var firestoreService: FirestoreService
    @State private var showAddSheet = false

    var body: some View {
        NavigationStack {
            List {
                ForEach(firestoreService.memos) { memo in
                    VStack(alignment: .leading, spacing: 4) {
                        Text(memo.title)
                            .font(.headline)
                        Text(memo.body)
                            .font(.subheadline)
                            .foregroundColor(.secondary)
                            .lineLimit(2)
                    }
                }
                .onDelete { indexSet in
                    for index in indexSet {
                        let memo = firestoreService.memos[index]
                        Task { try? await firestoreService.deleteMemo(memo) }
                    }
                }
            }
            .navigationTitle("メモ")
            .toolbar {
                Button { showAddSheet = true } label: {
                    Image(systemName: "plus")
                }
            }
            .sheet(isPresented: $showAddSheet) {
                AddMemoView()
            }
        }
    }
}
```

## 8. メモ追加画面

```swift
struct AddMemoView: View {
    @Environment(\.dismiss) private var dismiss
    @EnvironmentObject var firestoreService: FirestoreService

    @State private var title = ""
    @State private var body = ""
    @State private var isSaving = false

    private var canSave: Bool {
        !title.trimmingCharacters(in: .whitespaces).isEmpty
    }

    var body: some View {
        NavigationStack {
            Form {
                TextField("タイトル", text: $title)
                TextField("内容", text: $body, axis: .vertical)
                    .lineLimit(5...10)
            }
            .navigationTitle("メモを追加")
            .toolbar {
                ToolbarItem(placement: .cancellationAction) {
                    Button("キャンセル") { dismiss() }
                }
                ToolbarItem(placement: .confirmationAction) {
                    Button("保存") { save() }
                        .disabled(!canSave || isSaving)
                }
            }
        }
    }

    private func save() {
        isSaving = true
        Task {
            do {
                try await firestoreService.addMemo(
                    title: title.trimmingCharacters(in: .whitespaces),
                    body: body.trimmingCharacters(in: .whitespaces)
                )
                dismiss()
            } catch {
                isSaving = false
            }
        }
    }
}
```

## 9. ログアウト

```swift
struct ProfileView: View {
    @EnvironmentObject var authService: AuthService

    var body: some View {
        VStack(spacing: 24) {
            if let email = authService.currentUser?.email {
                Text(email)
                    .font(.headline)
            }

            Button(role: .destructive) {
                authService.signOut()
            } label: {
                Label("ログアウト", systemImage: "rectangle.portrait.and.arrow.right")
            }
        }
        .padding()
    }
}
```

## まとめ

| 項目 | 実装内容 |
|------|---------|
| 認証 | Firebase Auth（メール/パスワード） |
| DB | Cloud Firestore（`users/{uid}/memos/`） |
| モデル | `Codable` struct + `@DocumentID` |
| データ同期 | `addSnapshotListener`（リアルタイム） |
| 状態管理 | `@StateObject` + `environmentObject` |

Firebase Auth + Firestoreの組み合わせで、ユーザーごとにデータを分離したアプリが比較的少ないコード量で実装できます。`addSnapshotListener`によるリアルタイム同期は、明示的なリロード処理が不要になるため、UIの実装もシンプルになります。
