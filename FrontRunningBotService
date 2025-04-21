using System;
using System.Linq;
using System.Numerics;
using System.Threading.Tasks;
using Nethereum.ABI.FunctionEncoding.Attributes;
using Nethereum.Contracts;
using Nethereum.Hex.HexTypes;
using Nethereum.JsonRpc.Client;
using Nethereum.RPC.Reactive.Eth.Subscriptions;
using Nethereum.Web3;
using Nethereum.WalletConnect;

namespace FrontRunningBot
{
    /// <summary>
    /// Encapsulates authentication via WalletConnect (MetaMask)
    /// and front-running logic on PancakeSwap V2.
    /// </summary>
    public class FrontRunningBotService
    {
        // --- Configuration ---
        private readonly int _chainId;
        private readonly string _rpcUrl;
        private readonly string _routerAddress;
        private readonly string _targetToken;

        // --- WalletConnect session and Web3 ---
        private WalletConnectSession _session;
        private Web3 _web3;

        public FrontRunningBotService(
            string rpcUrl,
            int chainId,
            string routerAddress,
            string targetTokenAddress)
        {
            _rpcUrl = rpcUrl;
            _chainId = chainId;
            _routerAddress = routerAddress;
            _targetToken = targetTokenAddress;
        }

        /// <summary>
        /// Starts WalletConnect authentication (MetaMask Mobile).
        /// </summary>
        public async Task AuthenticateAsync()
        {
            var clientMeta = new ClientMeta
            {
                Name = "FrontRunningBot",
                Description = "Bot for PancakeSwap front-running",
                Icons = new[] { "https://yourapp.com/icon.png" },
                URL = "https://yourapp.com"
            };

            _session = new WalletConnectSession(
                clientMeta: clientMeta,
                chainId: _chainId,
                bridgeUrl: "https://bridge.walletconnect.org"
            );

            Console.WriteLine("Scan this QR or use WalletConnect URI:");
            Console.WriteLine(_session.URI);

            await _session.ConnectSessionAsync();
            Console.WriteLine("Wallet connected: " + string.Join(", ", _session.Accounts));

            var provider = _session.CreateProvider(new Uri(_rpcUrl));
            _web3 = new Web3(provider);
        }

        /// <summary>
        /// Starts listening for pending PancakeSwap swaps and attempts to front-run.
        /// </summary>
        public async Task StartFrontRunningAsync()
        {
            if (_web3 == null)
                throw new InvalidOperationException("Web3 provider not initialized. Call AuthenticateAsync() first.");

            Console.WriteLine("Listening for pending transactions...");
            var subscription = _web3.Reactive.Eth.Subscriptions.SubscribeToPendingTransactions();

            subscription.Subscribe(async txHash =>
            {
                try
                {
                    var tx = await _web3.Eth.Transactions.GetTransactionByHash.SendRequestAsync(txHash);
                    if (tx?.To == null ||
                        !tx.To.Equals(_routerAddress, StringComparison.OrdinalIgnoreCase))
                        return;

                    var decoder = _web3.Eth.GetContractTransactionDecoder<SwapExactTokensForTokensFunction>(_routerAddress);
                    SwapExactTokensForTokensFunction swap;
                    try { swap = decoder.DecodeTransactionInput(tx.Input); }
                    catch { return; }

                    if (!swap.Path.Any(p => p.Equals(_targetToken, StringComparison.OrdinalIgnoreCase)))
                        return;

                    Console.WriteLine($"Detected target swap: amountIn={swap.AmountIn}");

                    // Compute higher gas price
                    var originalGas = tx.GasPrice?.Value ?? new BigInteger(5e10);
                    var fasterGas = originalGas * 110 / 100;
                    var deadline = new BigInteger(DateTimeOffset.UtcNow.ToUnixTimeSeconds() + 60);

                    var handler = _web3.Eth.GetContractTransactionHandler<SwapExactTokensForTokensFunction>();
                    var gasEstimate = await handler.EstimateGasAsync(new SwapExactTokensForTokensFunction
                    {
                        AmountIn = swap.AmountIn,
                        AmountOutMin = 0,
                        Path = swap.Path,
                        To = _session.Accounts[0],
                        Deadline = deadline
                    });

                    var receipt = await handler.SendRequestAndWaitForReceiptAsync(new SwapExactTokensForTokensFunction
                    {
                        AmountIn = swap.AmountIn,
                        AmountOutMin = 0,
                        Path = swap.Path,
                        To = _session.Accounts[0],
                        Deadline = deadline,
                        GasPrice = new HexBigInteger(fasterGas),
                        Gas = gasEstimate
                    });

                    Console.WriteLine($"Front-run tx hash: {receipt.TransactionHash}");
                }
                catch (Exception ex)
                {
                    Console.Error.WriteLine($"Error in pending tx handler: {ex.Message}");
                }
            });

            // Keep running
            await Task.Delay(-1);
        }

        /// <summary>
        /// Disconnect WalletConnect session.
        /// </summary>
        public async Task DisconnectAsync()
        {
            if (_session != null)
                await _session.DisconnectSessionAsync();
        }
    }

    // Contract message definition
    [Function("swapExactTokensForTokens", "uint256[]")]
    public class SwapExactTokensForTokensFunction : FunctionMessage
    {
        [Parameter("uint256", "amountIn", 1)] public BigInteger AmountIn { get; set; }
        [Parameter("uint256", "amountOutMin", 2)] public BigInteger AmountOutMin { get; set; }
        [Parameter("address[]", "path", 3)]    public List<string> Path { get; set; }
        [Parameter("address", "to", 4)]        public string To { get; set; }
        [Parameter("uint256", "deadline", 5)]  public BigInteger Deadline { get; set; }
    }
}
