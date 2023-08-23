# 1. Merge stock and bond data time series 

```python
Monthly_CRSP_Universe = pd.merge(Monthly_CRSP_Bonds, Monthly_CRSP_Riskless, on=['Year', 'Month'], how='outer')
```

![image](https://github.com/demihe2004/Quantitative-Asset-Management/assets/135466801/ce88af8f-10e1-42de-86f6-4dda86f0482c)


# 2. Get vw/ 60/40/ unlevered risk parity/ levered risk parity portfolio return 

- vw
```python
# get vw return
Monthly_CRSP_Universe["total_mv"] = Monthly_CRSP_Universe['Stock_lag_MV'] + Monthly_CRSP_Universe['Bond_lag_MV']
Monthly_CRSP_Universe["Wt_stock"] = Monthly_CRSP_Universe['Stock_lag_MV']/Monthly_CRSP_Universe["total_mv"]
Monthly_CRSP_Universe["Wt_bond"] = 1 - Monthly_CRSP_Universe["Wt_stock"]
#check if any weight is >=1
any(Monthly_CRSP_Universe['Wt_stock'] >= 1)
Monthly_CRSP_Universe["Excess_Vw_Ret"] = Monthly_CRSP_Universe['Stock_Excess_Vw_Ret']*Monthly_CRSP_Universe["Wt_stock"]+Monthly_CRSP_Universe['Bond_Excess_Vw_Ret']*Monthly_CRSP_Universe["Wt_bond"]
```

- 60/40
```python
# get 60-40 portfolio return
Monthly_CRSP_Universe["Excess_60_40_Ret"] = Monthly_CRSP_Universe['Stock_Excess_Vw_Ret']*0.6+Monthly_CRSP_Universe['Bond_Excess_Vw_Ret']*0.4
```

- unlevered RP
``` python
for i in range(n,1164):
    subset_stock = Monthly_CRSP_Universe.iloc[i-n:i,Monthly_CRSP_Universe.columns.get_loc("Stock_Excess_Vw_Ret")]
    Monthly_CRSP_Universe.loc[i,"Stock_inverse_sigma_hat"] = 1/np.std(subset_stock)
    
    subset_bond = Monthly_CRSP_Universe.iloc[i-n:i,Monthly_CRSP_Universe.columns.get_loc("Bond_Excess_Vw_Ret")]
    Monthly_CRSP_Universe.loc[i,"Bond_inverse_sigma_hat"] = 1/np.std(subset_bond)
# get Unlevered k
Monthly_CRSP_Universe["Unlevered_k"] = 1/(Monthly_CRSP_Universe["Stock_inverse_sigma_hat"]+Monthly_CRSP_Universe["Bond_inverse_sigma_hat"])

# substitute inf
Monthly_CRSP_Universe = Monthly_CRSP_Universe.replace([np.inf, -np.inf], 0)

# get Excess_Unlevered_RP_Ret
Monthly_CRSP_Universe["Excess_Unlevered_RP_Ret"] = Monthly_CRSP_Universe["Unlevered_k"]*Monthly_CRSP_Universe["Stock_inverse_sigma_hat"]*Monthly_CRSP_Universe['Stock_Excess_Vw_Ret']+Monthly_CRSP_Universe["Unlevered_k"]*Monthly_CRSP_Universe["Bond_inverse_sigma_hat"]*Monthly_CRSP_Universe['Bond_Excess_Vw_Ret']
```

- levered RP
```python
# get Levered k
Monthly_CRSP_Universe["RP_ret"] = Monthly_CRSP_Universe['Stock_Excess_Vw_Ret']*Monthly_CRSP_Universe["Stock_inverse_sigma_hat"]+Monthly_CRSP_Universe["Bond_inverse_sigma_hat"]*Monthly_CRSP_Universe['Bond_Excess_Vw_Ret']
Monthly_CRSP_Universe_subset = Monthly_CRSP_Universe[(Monthly_CRSP_Universe["Year"] >= 1926) & (Monthly_CRSP_Universe["Year"] <= 2022)]

k_lev = np.std(Monthly_CRSP_Universe["Excess_Vw_Ret"])/np.std(Monthly_CRSP_Universe["RP_ret"])
Monthly_CRSP_Universe_subset["Levered_k"] = k_lev 
Monthly_CRSP_Universe_subset["Excess_Levered_RP_Ret"] = Monthly_CRSP_Universe_subset["Levered_k"]*(Monthly_CRSP_Universe_subset["Stock_inverse_sigma_hat"]*Monthly_CRSP_Universe_subset['Stock_Excess_Vw_Ret']+Monthly_CRSP_Universe_subset["Bond_inverse_sigma_hat"]*Monthly_CRSP_Universe_subset['Bond_Excess_Vw_Ret'])
```
![image](https://github.com/demihe2004/Quantitative-Asset-Management/assets/135466801/f583e5ba-c239-4a24-8594-a6a54652eb81)

# 3. Return analysis

- t-stat
``` python
X = sm.add_constant(np.ones(len(Port_Rets_subset)))
Y = Port_Rets_subset["Excess_Unlevered_RP_Ret"]*12
model = sm.OLS(Y, X)
results = model.fit()
replicate.iloc[4, 1] = results.tvalues

Y = Port_Rets_subset["Excess_Levered_RP_Ret"]*12
model = sm.OLS(Y, X)
results = model.fit()
replicate.iloc[5, 1] = results.tvalues
```
- 	Annualized Standard Deviation/ Annualized Sharpe Ratio/ Skewness/ Excess Kurtosis

![image](https://github.com/demihe2004/Quantitative-Asset-Management/assets/135466801/dd3d2ad6-ea01-4fa1-81d2-466e1299bbcc)
  
