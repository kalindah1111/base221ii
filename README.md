# base221ii
0x81b057dD6B2bAA9584aE5f129bAd27415A18D1E7
import time
from collections import defaultdict
from web3 import Web3

RPC_URL = "https://mainnet.base.org"

TRANSFER_TOPIC = Web3.keccak(
    text="Transfer(address,address,uint256)"
).hex()

WINDOW_BLOCKS = 25

ZERO = "0x0000000000000000000000000000000000000000"


def decode_address(topic):
    return "0x" + topic.hex()[-40:]


class WalletMetrics:

    def __init__(self):

        self.tokens = set()
        self.sent = 0
        self.received = 0
        self.first_block = None
        self.last_block = None

    def score(self):

        score = 0

        # Diversity
        score += min(len(self.tokens), 20)

        # Activity
        score += min(
            self.sent + self.received,
            50
        ) // 5

        # Persistence
        if (
            self.first_block is not None
            and self.last_block is not None
        ):

            lifetime = (
                self.last_block
                - self.first_block
            )

            score += min(
                lifetime // 20,
                20
            )

        return score


def main():

    w3 = Web3(
        Web3.HTTPProvider(RPC_URL)
    )

    if not w3.is_connected():
        raise RuntimeError("Cannot connect")

    print("Connected to Base")
    print("Building wallet reputation scores...\n")

    last_block = w3.eth.block_number

    wallets = defaultdict(WalletMetrics)

    while True:

        try:

            current_block = w3.eth.block_number

            if current_block >= last_block + WINDOW_BLOCKS:

                logs = w3.eth.get_logs({

                    "fromBlock":
                        current_block - WINDOW_BLOCKS,

                    "toBlock":
                        current_block,

                    "topics":
                        [TRANSFER_TOPIC]

                })

                for log in logs:

                    token = log["address"]

                    sender = decode_address(
                        log["topics"][1]
                    )

                    receiver = decode_address(
                        log["topics"][2]
                    )

                    block = log["blockNumber"]

                    if sender != ZERO:

                        wallet = wallets[sender]

                        wallet.tokens.add(token)
                        wallet.sent += 1

                        if wallet.first_block is None:
                            wallet.first_block = block

                        wallet.last_block = block

                    if receiver != ZERO:

                        wallet = wallets[receiver]

                        wallet.tokens.add(token)
                        wallet.received += 1

                        if wallet.first_block is None:
                            wallet.first_block = block

                        wallet.last_block = block

                ranking = sorted(

                    wallets.items(),

                    key=lambda x: x[1].score(),

                    reverse=True

                )[:20]

                print(
                    "\n===== Wallet Reputation =====\n"
                )

                for wallet, metrics in ranking:

                    print(wallet)

                    print(
                        "Score:",
                        metrics.score()
                    )

                    print(
                        "Tokens:",
                        len(metrics.tokens)
                    )

                    print(
                        "Received:",
                        metrics.received
                    )

                    print(
                        "Sent:",
                        metrics.sent
                    )

                    print()

                last_block = current_block

            time.sleep(3)

        except Exception as e:

            print(e)
            time.sleep(5)


if __name__ == "__main__":
    main()
