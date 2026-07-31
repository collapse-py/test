# Check-in GUI Implementation Plan

## Overview
Implement a 6x9 grid GUI for check-in system (54 days total) using SimpleGui library pattern from auctionhouse mod.

### Core Requirements
- **Grid**: 54 glass panes (6 rows × 9 columns), numbered 1-54 sequentially
- **Green pane**: Day already checked in - displays "X天" where X is the slot number
- **Yellow pane**: Today's check-in slot - allows player to click and sign in
- **Red pane**: Future/missed days - also display "X天" but red, non-interactive

---

## Design Decisions

### 1. UI Structure (Using `MenuType.GENERIC_9x6`)
```java
public class GUICheckIn extends SimpleGui {
    // Grid layout: slots 0-53 represent days 1-54
    
    @Override
    public void updateDisplay() {
        for each day slot (1 to 54):
            Determine color and interactivity based on state
            Create glass pane display element
    }
}
```

### 2. Day Numbering Logic
- **Day 1**: Server installation/mod activation date → grid position 0 (slot label "1天")
- **Today's offset**: Calculate how many days have passed since Day 1
- **Grid mapping**:
  - Slot index = dayOffset (capped at 53)
  - Name always shows `X天` where X is `(index + 1)` in Chinese numeral format

### 3. State Types
| Condition | Color (Item) | Interactive | Display Name |
|-----------|--------------|-------------|--------------|
| Day already checked in | `GREEN_STAINED_GLASS_PANE` | No | `X天` |
| Today's day | `YELLOW_STAINED_GLASS_PANE` | Yes | `今天` or current date |
| Past but missed/unchecked | `RED_STAINED_GLASS_PANE` | No | `X天` |
| Future not yet reached | `GREY_STAINED_GLASS_PANE` (similar to uncolored) | No | `X天` |

### 4. Reset Logic (After reaching Day 54)
- When player completes check-in for Day 54:
  - Recalculate day counter to reset progress
  - Clear stored date records associated with that player's past days

---

## Modifications Required

### File: `D:\Projet\mc\check-in\build.gradle`
**Action**: Add SimpleGui dependency (compileOnly)

```gradle
dependencies {
    // ... existing dependencies
    
    // SimpleGui - for server-side GUI library
    compileOnly include("eu.pb4:sgui:2.0.2+26.0")
}
```

### File: `D:\Projet\mc\check-in\src\main\java\com\collapse\checkin\CheckInData.java`
**Action**: Add methods to track check-ins and reset logic

```java
public class CheckInData extends SavedData {
    // Add method for tracking total check count:
    public int getCheckCount(UUID playerId) {
        String key = CHECK_COUNT_PREFIX + playerId.toString();
        return Integer.parseInt(checkIns.getOrDefault(key, "0"));
    }
    
    public void incrementCheckCount(UUID playerId) {
        int current = getCheckCount(playerId);
        checkIns.put(CHECK_COUNT_PREFIX + playerId.toString(), 
                     String.valueOf(current + 1));
        setDirty();
    }

    // Add method for getting per-day check-ins:
    public boolean hasCheckedInOnDate(UUID playerId, LocalDate date) {
        return false; // Implement logic to get stored dates
    }

    // Reset after reaching Day 54 should be called from GUI or command
    public void resetPlayerProgress(UUID playerId);
}
```

### File: `D:\Projet\mc\check-in\src\main\java\com\collapse\checkin\CheckInCommand.java`
**Action**: Register a new command `/checkin` that opens the GUI

```java
public class CheckInCommand {
    public static void register() {
        CommandRegistrationCallback.EVENT.register((dispatcher, registryAccess, environment) -> {
            dispatcher.register(Commands.literal("checkin")
                    .executes(context -> openGUI(context)));
        });
    }

    // Add method `openGUI`:
    private static int openGUI(CommandContext<CommandSourceStack> context) {
        ServerPlayer player = context.getSource().getPlayer();
        if (player == null) return Command.SINGLE_SUCCESS;
        
        new GUICheckIn(player).open();
        return Command.SINGLE_SUCCESS;
    }
}
```

### File: `D:\Projet\mc\check-in\src\main\java\com\collapse\checkin\gui\GUIAuctionHouse.java` (New)
**Action**: Create new GUI class (rename to proper package/class name)

- Use same import pattern as auctionhouse reference
- Implement `updateDisplay()` with loop from 0-53 (days 1-54)
- Use logic similar to:
  - For index 0-25: green pane if checked in for that day
  - For today's slot: yellow pane, interactive check-in on click
  - Others: red/grey panes based on date comparison

---

## Implementation Steps (Sequential)

1. **Add dependency** to `build.gradle` and sync project
2. **Update `CheckInData.java`**:
   - Add tracking for per-date check-ins (persist dates as strings)
   - Implement `resetPlayerProgress()` method
   
3. **Create GUI class `GUIAuctionHouse.java`** in `src/main/java/com/collapse/checkin/gui/` package:
   - Extend `SimpleGui` using `MenuType.GENERIC_9x6` (from auctionhouse reference)
   - Implement logic to:
     - Calculate which day slot corresponds to today's date
     - Render each of 54 slots with appropriate glass pane, name text, color
     - Handle click on yellow pane to trigger check-in

4. **Register command** in `CheckInCommand.java` to open the GUI

5. **Add missing items constants** to `GuiUtil` (or include directly):
   - `GREEN_STAINED_GLASS_PANE`, `YELLOW_STAINED_GLASS_PANE`, `RED_STAINED_GLASS_PANE`
   
6. **Build and test**:
   - Ensure mod compiles with all imports resolved
   - Open GUI via `/checkin` command → verify 54 panes displayed correctly
   - Click yellow pane on a valid day to check in and confirm UI updates

---

## Validation & Testing Plan

### Pre-Implementation Checks
- Verify SimpleGuiLib dependency resolves correctly (matches Java version)
- Confirm `MenuType.GENERIC_9x6` available in imported library

### In-Game Testing
1. Launch server + client with mod enabled
2. Start from Day 1 or recent date before first check-in attempt
3. Check GUI displays with:
   - Correct count (54 grid slots)
   - Each slot labeled "X天" where X follows sequence 1 → 54
   - Today's day highlighted as yellow, clickable

4. Interact with today's pane and verify:
   - Click sound plays
   - Check-in executes (money reward or whatever defined)
   - GUI closes automatically

5. Reopen GUI after check-in:
   - Past days show green + "X天" label (possibly still 1 day if missed previous)
   - Today's slot is now green/gray
   - Future slots remain red/grey

6. Simulate late-game by manually setting date forward or fast-forward logic:
   - Trigger reset after Day 54 completed → verify counter resets to zero

---

## Risk Assessment & Notes
- **SimpleGuiLib**: Must match Minecraft version (26.x) from dependencies. Default `MinecraftVersion` is likely 1.20/1.21 region based on auctionhouse config. Verify actual target version in gradle.properties file of check-in project.
- **Item Colors**: Using `Items.GREEN_STAINED_GLASS_PANE` via Minecraft items API (works server-side). Requires correct `getDefaultInstance()`.
- **Date calculations**: Use `LocalDate.now()` for today's calculation vs saved dates from server start. Ensure timezone consistency if needed.
