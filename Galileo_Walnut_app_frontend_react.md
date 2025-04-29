```bash
nano src/components/WalnutGame.js
```
```javascript
import React, { useState, useEffect, useCallback } from 'react';
import { createPublicClient, createWalletClient, http, custom } from 'viem';
import { defineChain } from 'viem';
import detectEthereumProvider from '@metamask/detect-provider';
import walnutABI from '../walnutABI.json';

const galileo = defineChain({
  id: 80087,
  name: 'Galileo',
  network: 'galileo',
  nativeCurrency: {
    decimals: 18,
    name: '0G Token',
    symbol: '0G',
  },
  rpcUrls: {
    default: { http: ['https://evmrpc-testnet.0g.ai'] },
  },
  blockExplorers: {
    default: { name: 'Chainscan', url: 'https://chainscan-galileo.0g.ai' },
  },
});

const WalnutGame = ({ account }) => {
  const [shellStrength, setShellStrength] = useState(0);
  const [round, setRound] = useState(0);
  const [kernel, setKernel] = useState(0);
  const [error, setError] = useState('');
  const [shakeAmount, setShakeAmount] = useState(1);
  const [networkError, setNetworkError] = useState('');
  const [walletClient, setWalletClient] = useState(null);

  const publicClient = createPublicClient({
    chain: galileo,
    transport: http('https://evmrpc-testnet.0g.ai'),
  });

  const contractAddress = '0xaa3f458dc680ea44c8b72a091dc18f8e5ccb38e8';

  // Khởi tạo walletClient khi account thay đổi
  useEffect(() => {
    const initWalletClient = async () => {
      const provider = await detectEthereumProvider();
      if (provider && account) {
        const client = createWalletClient({
          chain: galileo,
          transport: custom(provider),
          account,
        });
        setWalletClient(client);
      }
    };
    initWalletClient();
  }, [account]);

  const checkNetwork = async () => {
    try {
      const provider = await detectEthereumProvider();
      if (provider) {
        const chainId = await provider.request({ method: 'eth_chainId' });
        const normalizedChainId = parseInt(chainId, 16).toString();
        if (normalizedChainId !== '80087') {
          setNetworkError('Please switch to 0G Galileo Testnet (Chain ID: 80087)');
          return false;
        }
        setNetworkError('');
        return true;
      } else {
        setNetworkError('MetaMask not detected');
        return false;
      }
    } catch (err) {
      console.error('Network check error:', err);
      setNetworkError('Failed to check network: ' + (err.message || 'Unknown error'));
      return false;
    }
  };

  const refreshState = useCallback(async () => {
    if (!(await checkNetwork())) return;
    try {
      const strength = await publicClient.readContract({
        address: contractAddress,
        abi: walnutABI,
        functionName: 'getShellStrength',
      });
      const currentRound = await publicClient.readContract({
        address: contractAddress,
        abi: walnutABI,
        functionName: 'round',
      });
      setShellStrength(Number(strength));
      setRound(Number(currentRound));
      setError('');
    } catch (err) {
      console.error('Error fetching contract state:', err);
      setError('Failed to fetch contract state: ' + (err.message || 'Unknown error'));
    }
  }, [publicClient]);

  useEffect(() => {
    if (account) {
      refreshState();
    }
  }, [account, refreshState]);

  const handleHit = async () => {
    if (!(await checkNetwork())) return;
    if (!walletClient) {
      setError('Wallet client not initialized');
      return;
    }
    try {
      setError('');
      const { request } = await publicClient.simulateContract({
        address: contractAddress,
        abi: walnutABI,
        functionName: 'hit',
        account,
      });
      const hash = await walletClient.writeContract(request);
      await publicClient.waitForTransactionReceipt({ hash });
      refreshState();
    } catch (err) {
      console.error('Hit error:', err);
      setError(
        err.message.includes('SHELL_ALREADY_CRACKED')
          ? 'Shell is already cracked!'
          : 'Failed to hit: ' + (err.message || 'Unknown error')
      );
    }
  };

  const handleShake = async () => {
    if (!(await checkNetwork())) return;
    if (!walletClient) {
      setError('Wallet client not initialized');
      return;
    }
    try {
      setError('');
      const { request } = await publicClient.simulateContract({
        address: contractAddress,
        abi: walnutABI,
        functionName: 'shake',
        args: [shakeAmount],
        account,
      });
      const hash = await walletClient.writeContract(request);
      await publicClient.waitForTransactionReceipt({ hash });
      refreshState();
    } catch (err) {
      console.error('Shake error:', err);
      setError(
        err.message.includes('SHELL_ALREADY_CRACKED')
          ? 'Shell is already cracked!'
          : 'Failed to shake: ' + (err.message || 'Unknown error')
      );
    }
  };

  const handleReset = async () => {
    if (!(await checkNetwork())) return;
    if (!walletClient) {
      setError('Wallet client not initialized');
      return;
    }
    try {
      setError('');
      const { request } = await publicClient.simulateContract({
        address: contractAddress,
        abi: walnutABI,
        functionName: 'reset',
        account,
      });
      const hash = await walletClient.writeContract(request);
      await publicClient.waitForTransactionReceipt({ hash });
      refreshState();
    } catch (err) {
      console.error('Reset error:', err);
      setError(
        err.message.includes('SHELL_INTACT')
          ? 'Shell is still intact!'
          : 'Failed to reset: ' + (err.message || 'Unknown error')
      );
    }
  };

  const handleLook = async () => {
    if (!(await checkNetwork())) return;
    try {
      setError('');
      const result = await publicClient.readContract({
        address: contractAddress,
        abi: walnutABI,
        functionName: 'look',
        account,
      });
      setKernel(Number(result));
    } catch (err) {
      console.error('Look error:', err);
      setError(
        err.message.includes('NOT_A_CONTRIBUTOR')
          ? 'You are not a contributor this round!'
          : err.message.includes('SHELL_INTACT')
          ? 'Shell is still intact!'
          : 'Failed to look: ' + (err.message || 'Unknown error')
      );
    }
  };

  return (
    <div>
      <div className="status">
        <p>Shell Strength: {shellStrength}</p>
        <p>Round: {round}</p>
        <p>Kernel: {kernel}</p>
        {networkError && <p style={{ color: 'red' }}>{networkError}</p>}
        {error && <p style={{ color: 'red' }}>{error}</p>}
      </div>
      <div>
        <button onClick={handleHit} disabled={!account || shellStrength === 0}>
          Hit
        </button>
        <input
          type="number"
          value={shakeAmount}
          onChange={(e) => setShakeAmount(Number(e.target.value))}
          min="1"
          style={{ margin: '0 10px' }}
        />
        <button onClick={handleShake} disabled={!account || shellStrength === 0}>
          Shake
        </button>
        <button onClick={handleReset} disabled={!account || shellStrength > 0}>
          Reset
        </button>
        <button onClick={handleLook} disabled={!account}>
          Look
        </button>
      </div>
    </div>
  );
};

export default WalnutGame;
```
---

Nội dung file ``` src/components/WalletConnect.js ```
```bash
nano src/components/WalletConnect.js
```
```javascript
import React, { useEffect, useState } from 'react';
import detectEthereumProvider from '@metamask/detect-provider';

const WalletConnect = ({ setAccount }) => {
  const [error, setError] = useState('');

  const checkChainId = async (provider) => {
    const chainId = await provider.request({ method: 'eth_chainId' });
    console.log('Chain ID (raw):', chainId);

    // Chuyển chainId về dạng số thập phân để so sánh
    const normalizedChainId = parseInt(chainId, chainId.startsWith('0x') ? 16 : 10).toString();
    console.log('Normalized Chain ID:', normalizedChainId);

    if (normalizedChainId !== '80087') {
      setError('Please switch to 0G Galileo Testnet (Chain ID: 80087) in MetaMask.');
      return false;
    }

    setError('');
    return true;
  };

  const connectWallet = async () => {
    const provider = await detectEthereumProvider();
    if (provider) {
      try {
        const accounts = await provider.request({ method: 'eth_requestAccounts' });
        setAccount(accounts[0]);

        const isCorrectNetwork = await checkChainId(provider);
        if (!isCorrectNetwork) return;

      } catch (err) {
        console.error('Connect wallet error:', err);
        setError('Failed to connect wallet: ' + (err.message || 'Unknown error'));
      }
    } else {
      setError('Please install MetaMask!');
    }
  };

  useEffect(() => {
    const checkWallet = async () => {
      const provider = await detectEthereumProvider();
      if (provider) {
        try {
          const accounts = await provider.request({ method: 'eth_accounts' });
          if (accounts.length > 0) {
            setAccount(accounts[0]);
            await checkChainId(provider);
          }
        } catch (err) {
          console.error('Check wallet error:', err);
          setError('Failed to check wallet: ' + (err.message || 'Unknown error'));
        }
      }
    };
    checkWallet();
  }, [setAccount]);

  return (
    <div>
      <button onClick={connectWallet}>Connect Wallet</button>
      {error && <p className="error" style={{ color: 'red' }}>{error}</p>}
    </div>
  );
};

export default WalletConnect;
```
