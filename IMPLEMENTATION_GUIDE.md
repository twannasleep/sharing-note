# Multiple Chains Connector - Implementation Guide

## Table of Contents

1. [Project Setup](#1-project-setup)
2. [Core Implementation](#2-core-implementation)
3. [Chain Adapters](#3-chain-adapters)
4. [State Management](#4-state-management)
5. [UI Components](#5-ui-components)
6. [Integration Guide](#6-integration-guide)
7. [Testing](#7-testing)

## 1. Project Setup

### 1.1 Installation

```bash
# Create new project
pnpm create next-app my-web3-app
cd my-web3-app

# Install dependencies
pnpm add @mantine/core @mantine/hooks wagmi viem zustand @solana/web3.js
pnpm add -D typescript @types/react
```

### 1.2 Project Structure Setup

```bash
mkdir -p src/{core,adapters,store,components,hooks}
```

## 2. Core Implementation

### 2.1 Core Types (`src/core/types.ts`)

```typescript
export type ChainType = 'evm' | 'solana'

export interface Chain {
  id: string | number
  name: string
  icon?: string
  rpcUrls: string[]
  nativeCurrency: {
    name: string
    symbol: string
    decimals: number
  }
}

export interface ConnectorConfig {
  evm?: {
    chains: Chain[]
    connectors: string[]
  }
  solana?: {
    networks: string[]
    connectors: string[]
  }
}

export interface WalletInfo {
  address: string
  balance: string
  chainId: string | number
  ensName?: string
}
```

### 2.2 Constants (`src/core/constants.ts`)

```typescript
export const SUPPORTED_CHAINS = {
  evm: ['ethereum', 'polygon', 'optimism', 'arbitrum'],
  solana: ['mainnet-beta', 'devnet', 'testnet']
}

export const WALLET_CONNECTORS = {
  evm: ['metamask', 'walletconnect', 'coinbase'],
  solana: ['phantom', 'solflare']
}

export const DEFAULT_CHAINS = {
  evm: {
    ethereum: {
      id: 1,
      name: 'Ethereum',
      // ... other chain details
    },
    // ... other chains
  },
  solana: {
    'mainnet-beta': {
      id: '4sGjMW1sUnHzSxGspuhpqLDx6wiyjNtZ',
      name: 'Solana Mainnet',
      // ... other chain details
    }
  }
}
```

## 3. Chain Adapters

### 3.1 EVM Adapter (`src/adapters/evm/connector.ts`)

```typescript
import { configureChains, createConfig } from 'wagmi'
import { publicProvider } from 'wagmi/providers/public'

export class EVMAdapter implements ChainAdapter {
  private config: any
  
  constructor(chains: Chain[], connectors: string[]) {
    this.initializeConfig(chains, connectors)
  }

  private initializeConfig(chains: Chain[], connectors: string[]) {
    const { chains: configuredChains, publicClient } = configureChains(
      chains,
      [publicProvider()]
    )

    this.config = createConfig({
      autoConnect: true,
      connectors: this.getConnectors(connectors),
      publicClient,
    })
  }

  public async connect(connector: any): Promise<WalletInfo> {
    try {
      const result = await connector.connect()
      const balance = await this.getBalance(result.address)
      
      return {
        address: result.address,
        balance,
        chainId: result.chain.id,
        ensName: result.ensName
      }
    } catch (error) {
      throw new Error(`EVM connection failed: ${error.message}`)
    }
  }

  // ... implement other interface methods
}
```

### 3.2 Solana Adapter (`src/adapters/solana/connector.ts`)

```typescript
import { Connection, PublicKey } from '@solana/web3.js'

export class SolanaAdapter implements ChainAdapter {
  private connection: Connection
  
  constructor(endpoint: string) {
    this.connection = new Connection(endpoint)
  }

  public async connect(connector: any): Promise<WalletInfo> {
    try {
      await connector.connect()
      const publicKey = connector.publicKey.toString()
      const balance = await this.getBalance(publicKey)
      
      return {
        address: publicKey,
        balance,
        chainId: 'mainnet-beta'
      }
    } catch (error) {
      throw new Error(`Solana connection failed: ${error.message}`)
    }
  }

  // ... implement other interface methods
}
```

## 4. State Management

### 4.1 Store Implementation (`src/store/useWalletStore.ts`)

```typescript
import { create } from 'zustand'
import { persist } from 'zustand/middleware'

export const useWalletStore = create<WalletState>()(
  persist(
    (set, get) => ({
      // ... state implementation from technical proposal
    }),
    {
      name: 'wallet-storage',
      partialize: (state) => ({
        chainType: state.chainType,
        chain: state.chain
      })
    }
  )
)
```

## 5. UI Components

### 5.1 Connect Modal (`src/components/ConnectModal/index.tsx`)

```typescript
import { Modal, Stack, Button } from '@mantine/core'
import { useWalletStore } from '@/store/useWalletStore'

export const ConnectModal = () => {
  const { isModalOpen, setModalOpen, connect, chainType } = useWalletStore()
  const connectors = WALLET_CONNECTORS[chainType]

  return (
    <Modal
      opened={isModalOpen}
      onClose={() => setModalOpen(false)}
      title="Connect Wallet"
    >
      <Stack>
        {connectors.map((connector) => (
          <Button
            key={connector}
            onClick={() => connect(connector)}
          >
            {connector}
          </Button>
        ))}
      </Stack>
    </Modal>
  )
}
```

## 6. Integration Guide

### 6.1 Provider Setup (`src/providers/WalletProvider.tsx`)

```typescript
import { MultiChainConnector } from '@/core/connector'

const config = {
  evm: {
    chains: [mainnet, polygon],
    connectors: ['metamask', 'walletconnect'],
  },
  solana: {
    networks: ['mainnet-beta'],
    connectors: ['phantom']
  }
}

export const WalletProvider = ({ children }) => {
  const connector = new MultiChainConnector(config)
  
  return (
    <WalletContext.Provider value={connector}>
      {children}
    </WalletContext.Provider>
  )
}
```

### 6.2 App Integration (`pages/_app.tsx`)

```typescript
import { WalletProvider } from '@/providers/WalletProvider'
import { MantineProvider } from '@mantine/core'

function MyApp({ Component, pageProps }) {
  return (
    <MantineProvider>
      <WalletProvider>
        <Component {...pageProps} />
      </WalletProvider>
    </MantineProvider>
  )
}

export default MyApp
```

## 7. Testing

### 7.1 Unit Tests (`src/__tests__/adapters/evm.test.ts`)

```typescript
import { EVMAdapter } from '@/adapters/evm/connector'

describe('EVMAdapter', () => {
  let adapter: EVMAdapter

  beforeEach(() => {
    adapter = new EVMAdapter([/* test chains */], ['metamask'])
  })

  test('connects to wallet', async () => {
    const result = await adapter.connect(mockConnector)
    expect(result.address).toBeDefined()
    expect(result.balance).toBeDefined()
  })

  // ... more tests
})
```

### 7.2 Integration Tests (`src/__tests__/integration/wallet.test.ts`)

```typescript
import { renderHook, act } from '@testing-library/react-hooks'
import { useWallet } from '@/hooks/useWallet'

describe('Wallet Integration', () => {
  test('connects and updates state', async () => {
    const { result } = renderHook(() => useWallet())

    await act(async () => {
      await result.current.connect('metamask')
    })

    expect(result.current.isConnected).toBe(true)
    expect(result.current.address).toBeDefined()
  })
})
```

## Additional Resources

1. Example Repository: [github.com/your-org/multi-chain-connector-example](https://github.com/your-org/multi-chain-connector-example)
2. Documentation: [docs.your-domain.com/multi-chain-connector](https://docs.your-domain.com/multi-chain-connector)
3. API Reference: [docs.your-domain.com/api-reference](https://docs.your-domain.com/api-reference)

## Common Issues and Solutions

1. **Chain Switching Issues**
   - Always validate chain support before switching
   - Handle user rejection gracefully
   - Implement proper error messages

2. **Wallet Connection Errors**
   - Check if wallet extension is installed
   - Verify network connectivity
   - Handle timeout scenarios

3. **State Management**
   - Clear state on disconnect
   - Handle persistence errors
   - Implement proper cleanup
