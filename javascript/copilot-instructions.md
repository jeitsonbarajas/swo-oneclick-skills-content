# JavaScript Copilot Instructions

**Generated:** 2024-11-10  
**Version:** 1.0  
**Author:** SWO Team

## Overview
These are optimized GitHub Copilot instructions for JavaScript development, designed to enhance code generation quality and maintain consistency across projects.

## Best Practices

### Code Style
- Use modern ES6+ syntax (arrow functions, destructuring, template literals)
- Follow consistent naming conventions:
  - `camelCase` for variables and functions
  - `PascalCase` for classes and components
  - `UPPER_SNAKE_CASE` for constants
- Prefer `const` over `let`, avoid `var`
- Use meaningful, descriptive variable names

### Functions
- Keep functions small and focused (single responsibility)
- Use arrow functions for callbacks and functional programming
- Prefer pure functions when possible
- Add JSDoc comments for public APIs

### Error Handling
- Use try-catch blocks for async operations
- Provide meaningful error messages
- Handle edge cases explicitly
- Validate input parameters

### Async Operations
- Use `async/await` instead of promise chains
- Handle errors with try-catch in async functions
- Use `Promise.all()` for concurrent operations
- Avoid callback hell

### Testing
- Write unit tests for all functions
- Include edge cases and error scenarios
- Use descriptive test names
- Maintain high code coverage (>80%)

## Code Examples

### Function Example
```javascript
/**
 * Calculates the sum of an array of numbers
 * @param {number[]} numbers - Array of numbers to sum
 * @returns {number} The sum of all numbers
 */
const calculateSum = (numbers) => {
  if (!Array.isArray(numbers)) {
    throw new TypeError('Expected an array of numbers');
  }
  
  return numbers.reduce((sum, num) => sum + num, 0);
};
```

### Async/Await Example
```javascript
const fetchUserData = async (userId) => {
  try {
    const response = await fetch(`/api/users/${userId}`);
    
    if (!response.ok) {
      throw new Error(`HTTP error! status: ${response.status}`);
    }
    
    const data = await response.json();
    return data;
  } catch (error) {
    console.error('Failed to fetch user data:', error);
    throw error;
  }
};
```

## Project Structure
```
src/
├── components/     # Reusable UI components
├── utils/          # Utility functions
├── services/       # API and external services
├── config/         # Configuration files
├── constants/      # App-wide constants
└── tests/          # Test files
```

## Comments
- Use `//` for single-line comments
- Use `/** ... */` for JSDoc documentation
- Write self-documenting code when possible
- Explain "why" not "what" in comments

## Performance
- Avoid unnecessary re-renders
- Use debounce/throttle for frequent events
- Optimize loops and iterations
- Implement lazy loading when appropriate

---

*This file is part of SWO OneClick Skills - Created by SoftwareOne Team*
