# Experimental UI Feature Flag System

## Overview

The farming game now includes a comprehensive feature flag system that allows you to experiment with different UI elements while keeping both the current (classic) and experimental versions available. You can choose your experience directly from the Game Lobby.

**Note**: As of November 7, 2025, the **Default Experience (Barnyard)** has been promoted to the default UI for all new users. The Classic Experience remains available as an opt-out option.

## How It Works

### 1. UI Mode Selection

In the Game Lobby, you'll find a new **UI Experience Mode** section where you can choose between:

- **Classic Experience**: The original, stable interface (legacy opt-out)
- **Default Experience (Barnyard)**: The modern, farm-themed interface with enhanced features (now the default)

### 2. Granular Feature Control

When you select **Experimental UI**, you get fine-grained control over which experimental features to enable:

- 🏆 **Enhanced Scoreboard**: Improved layout with better data visualization and sorting
- ⚡ **Modern Action Panel**: Redesigned action interface with better UX and visual feedback
- 🎮 **Interactive Game Board**: Enhanced board with animations and improved player tracking
- 📝 **Advanced Game Log**: Rich formatting, filtering, and better event presentation

### 3. Persistent Settings

Your UI mode preferences are automatically saved to localStorage and persist across sessions, so you don't need to reconfigure them every time you play.

**Default Behavior**: New users (without existing localStorage preferences) will automatically receive the Default Experience (Barnyard) UI. Users with existing preferences will maintain their chosen experience.

## Current Experimental Features

### Experimental Scoreboard
- Beautiful gradient design with animations
- Card-based layout for each player
- Enhanced visual hierarchy
- Smooth hover effects and transitions
- Active player highlighting with shimmer effects
- Improved status badges and color coding

### Experimental Action Panel
- Enhanced visual wrapper with gradient background
- Clear labeling of experimental features
- Improved visual feedback

### Experimental Game Board
- Visual wrapper indicating experimental status
- Enhanced styling and presentation

### Experimental Game Log
- Rich formatting capabilities
- Improved event presentation
- Enhanced visual design

## For Developers

### Adding New Experimental Features

1. **Add to Store**: Update the `experimentalFeatures` interface in `gameStore.ts`
2. **Create Component**: Build your experimental component (see `ExperimentalScoreboard.tsx` as an example)
3. **Add Conditional Rendering**: Update the relevant view component with feature flag checks
4. **Update UI Selector**: Add the new feature to the `EXPERIMENTAL_FEATURES` array in `UIModeSelector.tsx`

### Example Implementation

```tsx
// In GameView.tsx
{uiMode === 'experimental' && experimentalFeatures.yourFeature ? (
  <YourExperimentalComponent {...props} />
) : (
  <YourClassicComponent {...props} />
)}
```

### Feature Flag Structure

```typescript
interface ExperimentalFeatures {
  scoreboard: boolean;
  actionPanel: boolean;
  gameBoard: boolean;
  gameLog: boolean;
  // Add new features here
}
```

## Best Practices

### For Users
- Start with one experimental feature at a time to isolate any issues
- Mix and match features to find your preferred experience
- Provide feedback on what works well and what doesn't

### For Developers
- Always provide a fallback to the classic component
- Use clear visual indicators when experimental features are active
- Keep experimental features clearly documented
- Ensure experimental features don't break core game functionality

## Technical Implementation

### Store Structure
- `uiMode`: 'classic' | 'experimental'
- `experimentalFeatures`: Object with boolean flags for each feature
- Automatic localStorage persistence
- Selector functions for easy state access

### Component Architecture
- Feature flags control conditional rendering
- Experimental components live in `/ui/` directory
- Clear separation between classic and experimental code paths
- No impact on core game logic

## Future Enhancements

The system is designed to be extensible. Future experimental features might include:

- Enhanced animations and transitions
- Different layout options
- Advanced filtering and sorting
- Custom themes and color schemes
- Performance optimizations
- Accessibility improvements

## Testing

To test the experimental UI system:

1. Go to the Game Lobby
2. Find the "UI Experience Mode" section
3. Toggle between Classic and Experimental modes
4. In Experimental mode, enable specific features
5. Join or create a game to see the changes in action
6. Refresh the page to verify settings persist

The feature flag system provides a safe way to experiment with new UI elements while maintaining stability and user choice. 