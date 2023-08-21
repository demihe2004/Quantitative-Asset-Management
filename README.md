# Quantitative-Asset-Management

**Project 1**
---
- Pull data from CRSP database and replicate market return time series available in Kenneth French website
- Calculate the equal-weighted market return, and the lagged total market capitalization
- output should be from January 1926 to December 2022, at a monthly frequency
- Calculate annulized mean/ standard deviation/ SR/ skewness/ excess kurtosis from return time series
- Compare and evaluate replication process by calculating correlation and max abs diff

**Project 2**
---
Replicate risk parity portfolio from Asness, Frazzini, and Pedersen (2012) and compared resturn with VW/ 60/40 portfolios using data from January 1926 to December 2022, at a monthly frequency
- Construct the equal-weighted bond market return, value-weighted bond market return, and lagged total bond market capitalization using CRSP Bond data 
- Aggregate stock, bond, and riskless datatables. For each year-month, calculate the lagged market value and excess value-weighted returns for both stocks and bonds 
- Calculate the monthly unlevered and levered risk-parity portfolio returns as defined. For the levered risk-parity portfolio, match the value-weighted portfolio’s ˆσ over the longest matched holding period of both
- Calculate stats and compare accross different portofolio construction method

**Project 3**
---
- Create bins according to size and value to replicate WML/ SMB factor strategies from Fama French
- Obtain parameters to define size (ME) and value (B/M ratio) from CRSP company data base and link signals to equity return database
- Divide bins and get value weighted return for each bin
- Create WML/ SMB signal by assigning factors to each company
- Calculate accumalated log excess return and other stats to compared with Fama French
