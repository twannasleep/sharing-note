# 🎨 Frontend Design Principles

> A comprehensive guide to design principles and patterns for modern frontend development.

---

## 📚 Table of Contents

1. [SOLID Principles](#solid-principles)
2. [Design Patterns](#design-patterns)
3. [Component Design](#component-design)
4. [State Management Patterns](#state-management-patterns)
5. [Code Organization](#code-organization)
6. [Performance Patterns](#performance-patterns)
7. [Error Handling](#error-handling)
8. [Best Practices](#best-practices)

---

## 🎯 SOLID Principles in Frontend

### 1. Single Responsibility Principle (SRP)

Each component or module should have one, and only one, reason to change.

```typescript
// ❌ Bad: Component doing too many things
const UserProfile = () => {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  const fetchUser = async () => {
    const response = await fetch('/api/user');
    const data = await response.json();
    setUser(data);
    setLoading(false);
  };

  const updateUser = async (userData) => {
    await fetch('/api/user', { method: 'PUT', body: JSON.stringify(userData) });
    await fetchUser();
  };

  return (
    <div>
      {loading ? <Spinner /> : <UserDetails user={user} onUpdate={updateUser} />}
    </div>
  );
};

// ✅ Good: Separated concerns
const useUser = () => {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  const fetchUser = async () => {
    const response = await fetch('/api/user');
    const data = await response.json();
    setUser(data);
    setLoading(false);
  };

  return { user, loading, fetchUser };
};

const UserProfile = () => {
  const { user, loading } = useUser();
  return (
    <div>
      {loading ? <Spinner /> : <UserDetails user={user} />}
    </div>
  );
};
```

### 2. Open-Closed Principle (OCP)

Software entities should be open for extension but closed for modification.

```typescript
// ❌ Bad: Hard to extend
const Button = ({ label, type }) => {
  if (type === 'primary') {
    return <button className="bg-blue">{label}</button>;
  } else if (type === 'secondary') {
    return <button className="bg-gray">{label}</button>;
  }
  return <button>{label}</button>;
};

// ✅ Good: Extensible through props
interface ButtonProps {
  label: string;
  variant?: 'primary' | 'secondary' | 'tertiary';
  size?: 'sm' | 'md' | 'lg';
  className?: string;
}

const Button: React.FC<ButtonProps> = ({
  label,
  variant = 'primary',
  size = 'md',
  className,
  ...props
}) => {
  const baseStyles = 'button';
  const variantStyles = `button--${variant}`;
  const sizeStyles = `button--${size}`;

  return (
    <button 
      className={clsx(baseStyles, variantStyles, sizeStyles, className)}
      {...props}
    >
      {label}
    </button>
  );
};
```

### 3. Liskov Substitution Principle (LSP)

Derived components should be substitutable for their base components.

```typescript
// Base Component Interface
interface CardProps {
  title: string;
  content: React.ReactNode;
}

// Base Card Component
const Card: React.FC<CardProps> = ({ title, content }) => (
  <div className="card">
    <h2>{title}</h2>
    <div>{content}</div>
  </div>
);

// Extended Components
const ProductCard: React.FC<CardProps & { price: number }> = ({
  title,
  content,
  price,
  ...props
}) => (
  <Card
    title={title}
    content={
      <>
        {content}
        <div className="price">${price}</div>
      </>
    }
    {...props}
  />
);
```

### 4. Interface Segregation Principle (ISP)

Components should not be forced to depend on interfaces they do not use.

```typescript
// ❌ Bad: Bloated interface
interface DataDisplayProps {
  data: any;
  loading: boolean;
  error: Error | null;
  onSort: () => void;
  onFilter: () => void;
  onExport: () => void;
  onPrint: () => void;
}

// ✅ Good: Segregated interfaces
interface DataProps {
  data: any;
  loading: boolean;
  error: Error | null;
}

interface SortableProps {
  onSort: () => void;
}

interface FilterableProps {
  onFilter: () => void;
}

interface ExportableProps {
  onExport: () => void;
}

const DataTable: React.FC<DataProps & Partial<SortableProps & FilterableProps>> = ({
  data,
  loading,
  error,
  onSort,
  onFilter
}) => {
  // Implementation
};
```

### 5. Dependency Inversion Principle (DIP)

High-level components should not depend on low-level components. Both should depend on abstractions.

```typescript
// ❌ Bad: Direct dependency
const UserList = () => {
  const [users, setUsers] = useState([]);

  useEffect(() => {
    fetch('/api/users')
      .then(res => res.json())
      .then(setUsers);
  }, []);

  return <List items={users} />;
};

// ✅ Good: Dependency injection
interface DataFetcher {
  fetch: () => Promise<any>;
}

const useData = (fetcher: DataFetcher) => {
  const [data, setData] = useState(null);

  useEffect(() => {
    fetcher.fetch().then(setData);
  }, [fetcher]);

  return data;
};

const UserList: React.FC<{ dataService: DataFetcher }> = ({ dataService }) => {
  const users = useData(dataService);
  return <List items={users} />;
};
```

---

## 🏗️ Design Patterns

### 1. Component Patterns

#### Compound Components

```typescript
// Button Group Pattern
const ButtonGroup = {
  Root: ({ children }: { children: React.ReactNode }) => (
    <div className="button-group">{children}</div>
  ),
  Button: ({ children }: { children: React.ReactNode }) => (
    <button className="button-group__item">{children}</button>
  ),
  Divider: () => <div className="button-group__divider" />,
};

// Usage
const Toolbar = () => (
  <ButtonGroup.Root>
    <ButtonGroup.Button>Copy</ButtonGroup.Button>
    <ButtonGroup.Divider />
    <ButtonGroup.Button>Paste</ButtonGroup.Button>
  </ButtonGroup.Root>
);
```

#### Render Props

```typescript
interface DataProviderProps<T> {
  fetchData: () => Promise<T>;
  children: (data: T) => React.ReactNode;
}

const DataProvider = <T,>({ fetchData, children }: DataProviderProps<T>) => {
  const [data, setData] = useState<T | null>(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchData().then(setData).finally(() => setLoading(false));
  }, [fetchData]);

  return children({ data, loading });
};

// Usage
const UserProfile = () => (
  <DataProvider fetchData={() => fetch('/api/user').then(r => r.json())}>
    {({ data, loading }) => (
      loading ? <Spinner /> : <UserDetails user={data} />
    )}
  </DataProvider>
);
```

#### Higher-Order Components (HOC)

```typescript
const withErrorBoundary = <P extends object>(
  WrappedComponent: React.ComponentType<P>,
  ErrorComponent: React.ComponentType<{ error: Error }>
) => {
  return class WithErrorBoundary extends React.Component<P, { error: Error | null }> {
    state = { error: null };

    static getDerivedStateFromError(error: Error) {
      return { error };
    }

    render() {
      if (this.state.error) {
        return <ErrorComponent error={this.state.error} />;
      }
      return <WrappedComponent {...this.props} />;
    }
  };
};
```

### 2. State Management Patterns

#### Container/Presenter Pattern

```typescript
// Container Component
const UserListContainer = () => {
  const [users, setUsers] = useState([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchUsers()
      .then(setUsers)
      .finally(() => setLoading(false));
  }, []);

  return <UserListPresenter users={users} loading={loading} />;
};

// Presenter Component
interface UserListPresenterProps {
  users: User[];
  loading: boolean;
}

const UserListPresenter: React.FC<UserListPresenterProps> = ({ users, loading }) => (
  <div>
    {loading ? (
      <Spinner />
    ) : (
      <ul>
        {users.map(user => (
          <li key={user.id}>{user.name}</li>
        ))}
      </ul>
    )}
  </div>
);
```

---

## 🎨 Component Design

### 1. Component Composition

```typescript
// Base Components
const Card = ({ children }: { children: React.ReactNode }) => (
  <div className="card">{children}</div>
);

const CardHeader = ({ children }: { children: React.ReactNode }) => (
  <div className="card-header">{children}</div>
);

const CardBody = ({ children }: { children: React.ReactNode }) => (
  <div className="card-body">{children}</div>
);

const CardFooter = ({ children }: { children: React.ReactNode }) => (
  <div className="card-footer">{children}</div>
);

// Usage
const ProductCard = ({ product }) => (
  <Card>
    <CardHeader>
      <h2>{product.name}</h2>
    </CardHeader>
    <CardBody>
      <img src={product.image} alt={product.name} />
      <p>{product.description}</p>
    </CardBody>
    <CardFooter>
      <Button onClick={() => addToCart(product)}>Add to Cart</Button>
    </CardFooter>
  </Card>
);
```

### 2. Style Organization

```typescript
// CSS Modules with TypeScript
import styles from './Button.module.css';

interface ButtonProps {
  variant: 'primary' | 'secondary';
  size: 'sm' | 'md' | 'lg';
}

const Button: React.FC<ButtonProps> = ({ variant, size, children }) => (
  <button
    className={clsx(
      styles.button,
      styles[`button--${variant}`],
      styles[`button--${size}`]
    )}
  >
    {children}
  </button>
);
```

---

## 🚀 Performance Patterns

### 1. Code Splitting

```typescript
// Route-based code splitting
const Routes = () => (
  <Switch>
    <Route path="/dashboard">
      <Suspense fallback={<Spinner />}>
        <DashboardPage />
      </Suspense>
    </Route>
    <Route path="/profile">
      <Suspense fallback={<Spinner />}>
        <ProfilePage />
      </Suspense>
    </Route>
  </Switch>
);

// Component-based code splitting
const Modal = lazy(() => import('./Modal'));
const Chart = lazy(() => import('./Chart'));
```

### 2. Memoization Patterns

```typescript
// Props memoization
const MemoizedComponent = memo(({ data }) => (
  <div>{data.map(item => <Item key={item.id} {...item} />)}</div>
), (prevProps, nextProps) => {
  return prevProps.data.length === nextProps.data.length;
});

// Callback memoization
const Component = () => {
  const handleClick = useCallback(() => {
    // Handle click
  }, []);

  return <Button onClick={handleClick}>Click me</Button>;
};

// Value memoization
const Component = ({ data }) => {
  const processedData = useMemo(() => {
    return expensiveOperation(data);
  }, [data]);

  return <div>{processedData}</div>;
};
```

---

## 🛡️ Error Handling

### 1. Error Boundaries

```typescript
class ErrorBoundary extends React.Component<
  { fallback: React.ReactNode },
  { hasError: boolean }
> {
  state = { hasError: false };

  static getDerivedStateFromError() {
    return { hasError: true };
  }

  componentDidCatch(error: Error, errorInfo: React.ErrorInfo) {
    logError(error, errorInfo);
  }

  render() {
    if (this.state.hasError) {
      return this.props.fallback;
    }

    return this.props.children;
  }
}
```

### 2. API Error Handling

```typescript
const useAPI = <T,>(url: string) => {
  const [state, setState] = useState<{
    data: T | null;
    loading: boolean;
    error: Error | null;
  }>({
    data: null,
    loading: true,
    error: null,
  });

  useEffect(() => {
    const fetchData = async () => {
      try {
        const response = await fetch(url);
        if (!response.ok) {
          throw new Error(`HTTP error! status: ${response.status}`);
        }
        const data = await response.json();
        setState({ data, loading: false, error: null });
      } catch (error) {
        setState({ data: null, loading: false, error });
      }
    };

    fetchData();
  }, [url]);

  return state;
};
```

---

## ✅ Best Practices Checklist

```markdown
□ Component Design
  □ Follow Single Responsibility Principle
  □ Use TypeScript for type safety
  □ Implement proper prop validation
  □ Create reusable components
  □ Document component API

□ State Management
  □ Choose appropriate state location
  □ Implement proper data flow
  □ Handle side effects correctly
  □ Manage form state effectively
  □ Cache API responses

□ Performance
  □ Implement code splitting
  □ Use proper memoization
  □ Optimize re-renders
  □ Lazy load components
  □ Monitor performance metrics

□ Error Handling
  □ Implement error boundaries
  □ Handle API errors gracefully
  □ Provide user feedback
  □ Log errors properly
  □ Implement fallbacks

□ Testing
  □ Write unit tests
  □ Implement integration tests
  □ Test error scenarios
  □ Test performance
  □ Test accessibility
```

---

Remember: Good design principles lead to maintainable, scalable, and efficient applications. Always consider the long-term implications of your design decisions.
