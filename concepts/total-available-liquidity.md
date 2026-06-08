# Total Available Liquidity

#### What is Total Available Liquidity?

**Total Available Liquidity (TAL)** represents the actual amount of funds or assets that can be actively traded or used **on-chain** at a reasonable price range.

On Native, **TAL** refers to the **total USD value paired** to a listed token.

* TAL is expressed in **USD value** and does not represent any specific underlying token.
* TAL only defines **one side of liquidity**: the **available liquidity to sell into** for the listed token.

#### How to calculate?

`TAL = HealthRatio * (Paired Liquidity USD + Bootstrapped Liquidity USD)`

* **Paired Liquidity USD**: The amount of USD value paired dedicatedly to the listed token
* **Bootstrapped Liquidity USD**: The amount of USD value provided by **market makers** for liquidity bootstrapping.

\
A tool to check your TAL: [Total Available Liquidity calculator](https://www.talcal.org/)
