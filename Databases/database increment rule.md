


head_increment  =  10 ^ subDigits
                =  always 10^3 = 1000  if sub always uses 3 digits
                =  10^2 = 100          if sub uses 2 digits (4-digit codes)

type_increment  =  10 ^ (totalDigits - 1)
                =  10^3 = 1000   for 4-digit
                =  10^4 = 10000  for 5-digit
                =  10^5 = 100000 for 6-digit

sub_increment   =  always 1

totalDigits > 4  →  subDigits = 3
totalDigits ≤ 4  →  subDigits = 2

typeDigits       →  always 1
headDigits       →  totalDigits - 1 - subDigits

typeIncrement    →  10 ^ (totalDigits - 1)
headIncrement    →  10 ^ subDigits
subIncrement     →  always 1


## 4-Digit (rule: 1 + 1 + 2)

TYPE      HEAD    SUB     TITLE
────────────────────────────────────────────────
1000              -       Assets
          1100    -       Bank
                  1101    NPR - NIC Asia
                  1102    NPR - Laxmi
          1200    -       Accounts Receivable
                  1201    General AR

2000              -       Liabilities
          2100    -       Payable
                  2101    Tax
                  2102    Insurance

3000              -       Equity / Net Assets
          3100    -       Retained Earnings
                  3101    General Fund

4000              -       Income
          4100    -       Program Revenue
                  4101    Grants

5000              -       Expenses
          5100    -       Salary
                  5101    Salary - Riya
          5200    -       Project 1
                  5201    Project 1 - Field Work

## 5-Digit (rule: 1 + 1 + 3)
TYPE       HEAD     SUB      TITLE
─────────────────────────────────────────────────
10000               -        Assets
           11000    -        Bank
                    11001    NPR - NIC Asia
                    11002    NPR - Laxmi
           12000    -        Accounts Receivable
                    12001    General AR

20000               -        Liabilities
           21000    -        Payable
                    21001    Tax
                    21002    Insurance

30000               -        Equity / Net Assets
           31000    -        Retained Earnings
                    31001    General Fund

40000               -        Income
           41000    -        Program Revenue
                    41001    Grants

50000               -        Expenses
           51000    -        Salary
                    51001    Salary - Riya
           52000    -        Project 1
                    52001    Project 1 - Field Work
                    
Max heads per type :   9   (11000–19000)
Max subs per head  : 999   (001–999)


## 6-Digit (rule: 1 + 2 + 3)

TYPE        HEAD      SUB       TITLE
──────────────────────────────────────────────────────
100000                          Assets        (type root)
            101000              Bank          (head 01)
                      101001    NIC Asia
                      101002    Laxmi
            102000              Accounts Receivable  (head 02)
                      102001    General AR
            103000              Fixed Assets  (head 03)
                      103001    Land
                      103002    Equipment

200000                          Liabilities   (type root)
            201000              Payable       (head 01)
                      201001    Tax
                      201002    Insurance
            202000              Loans         (head 02)
                      202001    Bank Loan

Max heads per type :  99   (110000–190000)
Max subs per head  : 999   (001–999)

4-digit  (1 + 1 + 2)
  type root  →  1000
  head 1     →  1100   (head digit = 1)
  head 2     →  1200   (head digit = 2)
  sub 1      →  1101
  sub 2      →  1102

5-digit  (1 + 1 + 3)
  type root  →  10000
  head 1     →  11000  (head digit = 1)
  head 2     →  12000  (head digit = 2)
  sub 1      →  11001
  sub 2      →  11002

6-digit  (1 + 2 + 3)
  type root  →  100000
  head 1     →  101000  (head digits = 01) ✅
  head 2     →  102000  (head digits = 02)
  sub 1      →  101001
  sub 2      →  101002

function firstHeadCode(typeRootCode, scheme) {
  // 4-digit: 1000 + 100  = 1100   (headIncrement = 100)
  // 5-digit: 10000 + 1000 = 11000  (headIncrement = 1000)
  // 6-digit: 100000 + 1000 = 101000 (headIncrement = 1000) ✅
  return typeRootCode + scheme.headIncrement
}

function nextHeadCode(existingHeadCodes, scheme) {
  const highest = Math.max(...existingHeadCodes)
  // 101000 + 1000 = 102000
  // 102000 + 1000 = 103000
  return highest + scheme.headIncrement
}





