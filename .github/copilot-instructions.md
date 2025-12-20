# Copilot instructions for WinWoL

WinWoL is a single WinUI 3 desktop application (solution: `WinWoL.sln`) with one UI project at `WinWoL/WinWoL`.
These instructions highlight the architecture, platform specifics, code patterns, and common workflows an AI coding agent should follow when making edits.

- **Big picture**: UI pages live in `WinWoL/WinWoL/Pages` and dialogs in `WinWoL/WinWoL/Pages/Dialogs` (XAML + code-behind). App lifecycle and the static window handle are in [App.xaml.cs](../WinWoL/WinWoL/App.xaml.cs#L1-L60) — `App.m_window` is created here and used wherever a native HWND is required. Business logic helpers are in `WinWoL/WinWoL/Methods`. Persistent data lives in `WinWoL/WinWoL/Datas/SQLiteHelper.cs` and models in `WinWoL/WinWoL/Models`.

**Build & Run (recommended)**
  - Preferred: Open `WinWoL.sln` in Visual Studio (recommended for WinUI, Windows App SDK and MSIX packaging).
    - `dotnet restore`
    - `dotnet build WinWoL.sln -c Debug`
    - To publish a self-contained build: `dotnet publish WinWoL/WinWoL.csproj -c Release -r win-x64 --self-contained`
  - Packaging + debug: this project uses Windows App SDK and MSIX packaging. The csproj has MSIX tooling enabled: see [WinWoL/WinWoL.csproj](../WinWoL/WinWoL/WinWoL.csproj#L1-L40). Use Visual Studio for packaging/signing and to create test installers.

- **Key dependencies** (see csproj): `Microsoft.WindowsAppSDK`, `Microsoft.Data.Sqlite`, `Newtonsoft.Json`, `Renci.SshNet` (SSH.NET), `PInvoke.User32`.

**Runtime / integration patterns to preserve**
  - File pickers require a window handle: the code attaches pickers with `WinRT.Interop.InitializeWithWindow` using the `App.m_window` HWND (examples: [WoLMethod.ExportConfig](../WinWoL/WinWoL/Methods/WoLMethod.cs#L1-L120), [SSHMethod.ExportConfig](../WinWoL/WinWoL/Methods/SSHMethod.cs#L1-L60)). Avoid invoking pickers before the window exists.
  - UI updates from background threads use `DispatcherQueue.TryEnqueue` (see [WoL.xaml.cs](../WinWoL/WinWoL/Pages/WoL.xaml.cs#L1-L40)). Maintain UI-thread marshalling when modifying controls.
  - `SQLiteHelper` uses a relative `Data Source=wol.db` and runs schema checks in the constructor and `UpgradeDatabase` (see [SQLiteHelper.cs](../WinWoL/WinWoL/Datas/SQLiteHelper.cs#L1-L60)). Use this helper for all DB access and add migration SQL here when adding columns or changing schemas.
  - Dialogs are shown using `ContentDialog` patterns with `.XamlRoot` set to the current page, `Style = Application.Current.Resources["DefaultContentDialogStyle"]`, and `DefaultButton` configured (see [AddWoL.xaml.cs](../WinWoL/WinWoL/Pages/Dialogs/AddWoL.xaml.cs#L1-L40) and [AddSSH.xaml.cs](../WinWoL/WinWoL/Pages/Dialogs/AddSSH.xaml.cs#L1-L40)).

**Common code patterns**
  - Utility classes (`WoLMethod`, `SSHMethod`, `GeneralMethod`) use static members and are invoked directly from pages. Preserve these signatures unless there's a clear reason to change to instance-based services.
  - Import/export: file flows use `Newtonsoft.Json` + `FileOpenPicker`/`FileSavePicker` with `CachedFileManager.DeferUpdates` and `CompleteUpdatesAsync` (see [WoLMethod.ExportConfig](../WinWoL/WinWoL/Methods/WoLMethod.cs#L40-L140) and [SSHMethod.ExportConfig](../WinWoL/WinWoL/Methods/SSHMethod.cs#L1-L80)). Note: saving to synced folders (OneDrive) may cause `DeferUpdates` issues — keep existing user-facing warnings.
  - Localization: strings are fetched with `Windows.ApplicationModel.Resources.ResourceLoader` from `.resw` files in `WinWoL/WinWoL/Language/*`. Follow this pattern when adding UI text; prefer resource keys where possible (see [WoL.xaml.cs](../WinWoL/WinWoL/Pages/WoL.xaml.cs#L1-L20)).

**Pitfalls and gotchas**
  - `CachedFileManager` calls can return unexpected results on OneDrive; keep user-facing text and don't rely on the returned status for critical logic.
  - The codebase often uses `Thread` rather than `Task`/`async` for background work and relies on `DispatcherQueue.TryEnqueue` to update UI. When refactoring to `Task`/`async`, preserve UI dispatch behavior and test concurrency interactions.
  - `wol.db` is a relative path and behaves differently when running as a packaged MSIX app vs VS debug; test DB and file access flows under both contexts.
  - Data re-ordering is implemented by swapping row `Id` fields in `SQLiteHelper` (`UpSwapRows` / `DownSwapRows`). Be careful when changing primary-key semantics.
  - The app enforces single-instance behavior in [App.xaml.cs](../WinWoL/WinWoL/App.xaml.cs#L1-L90) using `AppInstance` and `SetForegroundWindow` via `PInvoke.User32` to focus the existing window.

**Where to start for common tasks**
  - Add a new dialog: copy `Pages/Dialogs/AddWoL.xaml(.cs)` pattern, use `.XamlRoot` and `Application.Current.Resources["DefaultContentDialogStyle"]`, set `DefaultButton`, and wire up data model binding similarly to existing dialogs.
  - Add a DB column or change schema: update `SQLiteHelper.CreateTableIfNotExists`, and include migration SQL in `UpgradeDatabase` and `UpgradeDatabaseVersion`. Use `GetDatabaseVersion()` to gate migrations, and keep `Data Source=wol.db` awareness in mind.
  - Add a new background operation that touches UI: use `Task.Run` or `Thread` and always call `DispatcherQueue.TryEnqueue` to update UI. If using file pickers from background flows, ensure you initialize the picker with `App.m_window`.

**No automated tests found**: there are no unit tests. For UI or platform changes, test iteratively in Visual Studio: run the packaged version and the Debug/sideloaded app.

If anything is unclear or you want more examples (e.g., exact lines for a DB migration, sample dialog changes, or the single-instance behavior), tell me which task and I will iterate this file.

If any section is unclear or you want me to expand examples (e.g., exact lines to change for a DB migration or a sample dialog implementation), tell me which task and I will iterate this file.
