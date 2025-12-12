# Copilot instructions for WinWoL

This project is a single WinUI3 desktop app (solution: `WinWoL.sln`) with one main project at `WinWoL/WinWoL`. The guidance below highlights architecture, common patterns, build/run steps, and examples an AI coding agent should follow when editing or extending the codebase.

- **Big picture**: UI lives in `WinWoL/WinWoL/Pages` and `Pages/Dialogs` (XAML + code-behind). App startup and window handle are in `WinWoL/WinWoL/App.xaml.cs` (important: `App.m_window` is created here). Business logic helpers are plain C# classes in `WinWoL/WinWoL/Methods`. Persistent data is via a local SQLite DB managed by `WinWoL/WinWoL/Datas/SQLiteHelper.cs` and models live in `WinWoL/WinWoL/Models`.

- **Build & run (recommended)**
  - Preferred: Open `WinWoL.sln` in Visual Studio (recommended for WinUI + MSIX tooling).
   # Copilot instructions for WinWoL

   Short, actionable notes to help an AI coding agent be productive quickly.

   - **Project shape**: single WinUI3 desktop app. Main project: `WinWoL/WinWoL` (solution: `WinWoL.sln`). UI XAML+code-behind live in `WinWoL/WinWoL/Pages` and dialogs in `WinWoL/WinWoL/Pages/Dialogs`.

   - **Startup & window handle**: `WinWoL/WinWoL/App.xaml.cs` creates a single instance and assigns `App.m_window` (used elsewhere). Do NOT invoke file pickers or WindowNative calls before `App.m_window` is created in `OnLaunched`.
     - Example: `WinRT.Interop.WindowNative.GetWindowHandle(App.m_window)` is used in `Methods/WoLMethod.cs` and `Methods/SSHMethod.cs` before calling `InitializeWithWindow`.

   - **Single-instance & window behavior**: App uses `AppInstance.FindOrRegisterForKey(...)` to redirect activations. `App.xaml.cs` also uses `PInvoke.User32` to restore/set foreground and a DPI-aware `SetWindowSize` helper; keep this behavior when modifying startup.

   - **File pickers & saved files**: All file pickers are UWP pickers initialized with `WinRT.Interop.InitializeWithWindow`. Save flows call `CachedFileManager.DeferUpdates` / `CompleteUpdatesAsync` — these can fail on OneDrive/synced folders; code already returns localized warnings.

   - **Data layer**: `Datas/SQLiteHelper.cs` is the single DB helper. Connection string: `Data Source=wol.db` (relative path). Schema creation is in `CreateTableIfNotExists`; migrations run via `UpgradeDatabase()` and version bookkeeping with `UpgradeDatabaseVersion()` / `GetDatabaseVersion()`. Always use this helper for DB operations to preserve schema and migration logic.

   - **Common code patterns**:
     - Business logic helpers are static-style classes under `Methods/` (e.g., `WoLMethod`, `SSHMethod`, `GeneralMethod`).
     - Models live in `Models/` (`WoLModel`, `SSHModel`, `SSHPasswdModel`).
     - Localization: strings read via `Windows.ApplicationModel.Resources.ResourceLoader` and `.resw` files under `Language/*`.
     - Dialog pattern: create dialog instance, set `dialog.XamlRoot = this.XamlRoot`, `dialog.Style = Application.Current.Resources["DefaultContentDialogStyle"]`, set button text from `ResourceLoader`, then `await dialog.ShowAsync()` (see `Pages/WoL.xaml.cs` and `Pages/Dialogs/AddWoL.xaml.cs`).

   - **Threading & UI updates**: many pages spawn raw `Thread` and update UI via `DispatcherQueue.GetForCurrentThread()` + `_dispatcherQueue.TryEnqueue(...)`. When adding background work prefer `Task.Run` or follow the existing `Thread` + `TryEnqueue` pattern — ensure UI work runs on the UI Dispatcher.

   - **Export/import format**: JSON using `Newtonsoft.Json`. Files use extensions `.wolconfigx` and `.sshconfigx` (see `Methods/WoLMethod.cs` and `Methods/SSHMethod.cs`).

   - **Build / package / debug**:
     - Recommended: open `WinWoL.sln` in Visual Studio (best for WinUI, MSIX, and debugging).
     - CLI quick commands:
       - `dotnet restore`
       - `dotnet build WinWoL.sln -c Debug`
       - `dotnet publish WinWoL/WinWoL.csproj -c Release -r win-x64 --self-contained`
     - Packaging: app uses Windows App SDK / MSIX (`WinWoL/WinWoL.csproj` and `Package.appxmanifest`) — use Visual Studio packaging tooling for signing/store flows.

   - **Key dependencies**: `Microsoft.WindowsAppSDK`, `Microsoft.Data.Sqlite`, `Newtonsoft.Json`, `SSH.NET`, `PInvoke.User32`.

   - **Where to start for common tasks (practical examples)**
     - Add a dialog: copy `WinWoL/WinWoL/Pages/Dialogs/AddWoL.xaml(.cs)` and follow the `.XamlRoot` + `Style` pattern.
     - Add a DB column: update `CreateTableIfNotExists` and add migration SQL inside `UpgradeDatabase()`; maintain version logic in `UpgradeDatabaseVersion()` / `GetDatabaseVersion()`.
     - Add a file picker/save: call `WinRT.Interop.WindowNative.GetWindowHandle(App.m_window)` and `WinRT.Interop.InitializeWithWindow.Initialize(picker, hWnd)` before Pick/Save.

   - **Pitfalls**
     - Don’t call pickers before `App.m_window` exists.
     - `CachedFileManager` flows can behave oddly on OneDrive — tests should cover save/load from synced folders.
     - DB file path is relative — packaged vs VS run contexts differ; test both.

   - **Tests**: there are no automated tests in this repository. Make small, runnable changes and test in Visual Studio.

   If you want, I can expand any section with concrete line-level examples (DB migration snippet, dialog scaffolding, or a safe Task-based refactor of a threaded method). Please tell me which example you want next.
