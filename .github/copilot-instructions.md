# Copilot Workspace Instructions

## Project Overview

**Soc Ops** is a Social Bingo web app built with **Blazor WebAssembly (.NET 10)**. Players find people who match icebreaker questions to mark squares and get 5 in a row.

## Key Commands

```bash
dotnet build SocOps/SocOps.csproj          # Build (must pass before committing)
dotnet run --project SocOps/SocOps.csproj  # Dev server → http://localhost:5166
```

## Architecture & Data Flow

`Home.razor` is the **single smart container** — it injects `BingoGameService`, wires `OnStateChanged` → `StateHasChanged`, and subscribes/unsubscribes in `OnInitializedAsync`/`Dispose`. All other components are **dumb** (Parameters + EventCallbacks only).

Click event chain: `BingoSquare` → `BingoBoard.OnSquareClick` → `GameScreen.OnSquareClick` → `Home.razor` → `GameService.HandleSquareClick()` → `NotifyStateChanged()` → re-render.

```
Services/
  BingoGameService.cs    # Scoped DI — owns all mutable state, persists to localStorage
  BingoLogicService.cs   # All static methods — board generation, toggle, win detection
Models/
  GameState.cs           # Enum: Start → Playing → Bingo
  BingoSquareData.cs     # Id, Text, IsMarked, IsFreeSpace
  BingoLine.cs           # Winning line (type, index, square ids)
Data/
  Questions.cs           # Edit here to change bingo content (needs 24+ strings)
```

## State Management

- `BingoGameService` is the only DI-registered service (`AddScoped` in `Program.cs`)
- `BingoLogicService` is **not injected** — called via static methods from `BingoGameService`
- State is persisted to `localStorage` via `IJSRuntime` with a `STORAGE_VERSION` int — bump it to invalidate all saved games
- `_ = SaveGameStateAsync()` is intentionally fire-and-forget (no `await`)

## Board Specifics

- Always 5×5 = 25 squares; **index 12 is always FREE SPACE** (`IsFreeSpace = true`, pre-marked)
- `BingoLogicService.GenerateBoard()` shuffles `Questions.QuestionsList` (Fisher-Yates), takes 24
- `ToggleSquare` returns a **new list** (immutable update pattern) — never mutate board in place

## Styling

Custom utility classes (Tailwind-like) defined in `wwwroot/css/app.css` — no external CSS framework.

- Layout: `.flex`, `.flex-col`, `.grid`, `.grid-cols-5`, `.items-center`, `.justify-between`
- Color tokens: `.bg-accent` (primary blue), `.bg-marked` / `.border-marked-border` (marked green)
- Conditional CSS: use a `GetCssClasses()` method in `@code` block (see `BingoSquare.razor`)
- Add new utilities directly to `wwwroot/css/app.css` following existing patterns
- Use CSS variables in `:root` for theming

## Component Conventions

- Components declare `[Parameter]` props and `EventCallback` outputs — no direct service injection
- `BingoSquare.razor` shows the pattern for computed conditional classes via `GetCssClasses()`
- `aria-pressed` and `aria-label` are set on interactive squares — maintain accessibility attributes
- `@implements IDisposable` + unsubscribe `OnStateChanged` in `Dispose()` is required on any component that subscribes to service events
