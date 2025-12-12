# Copilot instructions for WinWoL

This project is a single WinUI3 desktop app (solution: `WinWoL.sln`) with one main project at `WinWoL/WinWoL`. The guidance below highlights architecture, common patterns, build/run steps, and examples an AI coding agent should follow when editing or extending the codebase.

- **Big picture**: UI lives in `WinWoL/WinWoL/Pages` and `Pages/Dialogs` (XAML + code-behind). App startup and window handle are in `WinWoL/WinWoL/App.xaml.cs` (important: `App.m_window` is created here). Business logic helpers are plain C# classes in `WinWoL/WinWoL/Methods`. Persistent data is via a local SQLite DB managed by `WinWoL/WinWoL/Datas/SQLiteHelper.cs` and models live in `WinWoL/WinWoL/Models`.

- **Build & run (recommended)**
  - Preferred: Open `WinWoL.sln` in Visual Studio (recommended for WinUI + MSIX tooling).
  - CLI (quick):
    - `dotnet restore`
    - `dotnet build WinWoL.sln -c Debug`
    - To publish a self-contained build: `dotnet publish WinWoL/WinWoL.csproj -c Release -r win-x64 --self-contained`
  - Packaging: the project uses MSIX/Windows App SDK (see `WinWoL/WinWoL.csproj`) — use Visual Studio packaging UI for signing and store flows.

- **Key dependencies** (from `WinWoL/WinWoL.csproj`): `Microsoft.WindowsAppSDK`, `Microsoft.Data.Sqlite`, `Newtonsoft.Json`, `SSH.NET`, `PInvoke.User32`.

- **Runtime / integration patterns you must preserve**
  - File pickers require a window handle: the code expects `App.m_window` to be set and uses `WinRT.Interop.InitializeWithWindow` (examples: `WinWoL/WinWoL/Methods/WoLMethod.cs`, `WinWoL/WinWoL/Methods/SSHMethod.cs`). Avoid calling picker code before `App.m_window` exists.
  - UI updates from background threads use `DispatcherQueue.TryEnqueue` (see `WinWoL/WinWoL/Pages/WoL.xaml.cs`). Keep UI-thread marshalling when modifying controls.
  - Database helper (`SQLiteHelper`) opens a local SQLite file (`Data Source=wol.db`) and runs schema upgrades in ctor/`UpgradeDatabase`. Use that class for all DB reads/writes to keep schema logic centralized.
  - Dialogs are instantiated and configured with `.XamlRoot`, `Style = Application.Current.Resources["DefaultContentDialogStyle"]`, and `DefaultButton` (see `Pages/*` dialogs). Follow the same pattern when adding dialogs.

- **Common code patterns**
  - Methods classes often use static methods for utilities (e.g., `WoLMethod`, `SSHMethod`, `GeneralMethod`). Prefer preserving signature style when adding helpers.
  - Serialization for import/export uses `Newtonsoft.Json` to read/write `.wolconfigx`/`.sshconfigx` files (see `WoLMethod.ExportConfig` / `ImportConfig`).
  - Localization uses `.resw` resource files and `Windows.ApplicationModel.Resources.ResourceLoader` for strings (see `Pages/WoL.xaml.cs` and `Language/*` folders).

- **Pitfalls and gotchas**
  - File save/open flows call `CachedFileManager.DeferUpdates` and `CompleteUpdatesAsync` — these can behave unexpectedly on OneDrive or synced folders (code already returns warnings).
  - Some code uses raw `Thread` rather than `Task`/`async` for background work; be conservative when refactoring threading to preserve UI timing and error handling.
  - `wol.db` path is relative; behavior differs when running from Visual Studio vs packaged app — test DB flows in both contexts.

- **Where to start for common tasks**
  - Add a new dialog: copy `Pages/Dialogs/AddWoL.xaml(.cs)` pattern and wire it in the calling page using the same `.XamlRoot` and style setup.
  - Add a DB column: update `SQLiteHelper.CreateTableIfNotExists`, add migration logic in `UpgradeDatabase` and keep `GetDatabaseVersion`/`UpgradeDatabaseVersion` semantics.
  - Add a new background operation that touches UI: run work on a background thread or `Task.Run`, then `DispatcherQueue.TryEnqueue` to update the UI.

- **No automated tests found**: there are no unit tests in the repo. Prefer incremental manual testing via Visual Studio when changing UI or platform integrations.

If any section is unclear or you want me to expand examples (e.g., exact lines to change for a DB migration or a sample dialog implementation), tell me which task and I will iterate this file.
