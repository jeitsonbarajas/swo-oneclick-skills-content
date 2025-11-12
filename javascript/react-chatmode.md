# React Development Chat Mode

**Generated:** 2025-11-09  
**Version:** 1.0  
**Technology:** React + JavaScript  
**Author:** SWO Team

## Purpose
Specialized chat mode configuration for React component development with GitHub Copilot Chat. This mode provides context-aware assistance for building modern React applications.

## Features
- Component architecture guidance
- State management best practices
- Hooks usage patterns
- Performance optimization tips
- Testing strategies for React components

## React Best Practices

### Component Structure
```jsx
import React, { useState, useEffect } from 'react';
import PropTypes from 'prop-types';

/**
 * UserProfile component displays user information
 * @param {Object} props - Component props
 * @param {string} props.userId - The user ID to display
 * @param {Function} props.onUpdate - Callback when profile updates
 */
const UserProfile = ({ userId, onUpdate }) => {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    fetchUserData();
  }, [userId]);

  const fetchUserData = async () => {
    try {
      setLoading(true);
      const data = await getUserById(userId);
      setUser(data);
    } catch (err) {
      setError(err.message);
    } finally {
      setLoading(false);
    }
  };

  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error}</div>;
  if (!user) return null;

  return (
    <div className="user-profile">
      <h2>{user.name}</h2>
      <p>{user.email}</p>
    </div>
  );
};

UserProfile.propTypes = {
  userId: PropTypes.string.isRequired,
  onUpdate: PropTypes.func
};

export default UserProfile;
```

### Hooks Guidelines

#### useState
- Initialize with proper default values
- Use functional updates for derived state
- Don't store derived values in state

#### useEffect
- Specify dependencies accurately
- Clean up subscriptions and timers
- Avoid unnecessary re-renders

#### useCallback / useMemo
- Memoize expensive calculations
- Prevent unnecessary child re-renders
- Use sparingly - measure before optimizing

### State Management
- Use local state for component-specific data
- Lift state up when sharing between components
- Consider Context API for global state
- Use Redux/Zustand for complex state logic

### File Organization
```
src/
├── components/
│   ├── common/          # Reusable components
│   ├── features/        # Feature-specific components
│   └── layout/          # Layout components
├── hooks/               # Custom hooks
├── context/             # React Context providers
├── utils/               # Utility functions
└── styles/              # Global styles
```

### Performance Tips
1. Use React.memo for expensive components
2. Implement code splitting with React.lazy
3. Optimize re-renders with useCallback/useMemo
4. Use React DevTools Profiler
5. Virtualize long lists (react-window)

### Testing
```jsx
import { render, screen, fireEvent } from '@testing-library/react';
import UserProfile from './UserProfile';

describe('UserProfile', () => {
  it('displays user name', async () => {
    render(<UserProfile userId="123" />);
    
    const userName = await screen.findByText('John Doe');
    expect(userName).toBeInTheDocument();
  });

  it('handles loading state', () => {
    render(<UserProfile userId="123" />);
    
    expect(screen.getByText('Loading...')).toBeInTheDocument();
  });
});
```

## Common Patterns

### Custom Hook
```jsx
const useUserData = (userId) => {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    let cancelled = false;

    const fetchData = async () => {
      try {
        const result = await fetchUser(userId);
        if (!cancelled) {
          setData(result);
        }
      } catch (err) {
        if (!cancelled) {
          setError(err);
        }
      } finally {
        if (!cancelled) {
          setLoading(false);
        }
      }
    };

    fetchData();

    return () => {
      cancelled = true;
    };
  }, [userId]);

  return { data, loading, error };
};
```

### Context Provider
```jsx
const ThemeContext = createContext();

export const ThemeProvider = ({ children }) => {
  const [theme, setTheme] = useState('light');

  const toggleTheme = () => {
    setTheme(prev => prev === 'light' ? 'dark' : 'light');
  };

  return (
    <ThemeContext.Provider value={{ theme, toggleTheme }}>
      {children}
    </ThemeContext.Provider>
  );
};

export const useTheme = () => {
  const context = useContext(ThemeContext);
  if (!context) {
    throw new Error('useTheme must be used within ThemeProvider');
  }
  return context;
};
```

## Accessibility
- Use semantic HTML elements
- Add proper ARIA labels
- Ensure keyboard navigation
- Test with screen readers
- Maintain proper heading hierarchy

---

*This chat mode is part of SWO OneClick Skills - Created by SoftwareOne Team*
