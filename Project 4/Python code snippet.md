# 1. Lagged variance weight for each factor 
```python
for i in range(len(factor)):
    # get lagged var of past 22 days
    ff_d['vol'] = ff_d[factor[i]].rolling(window=22).var()
    
    v_factor = ff_d.groupby('month').agg(var=('vol', lambda x: x.iloc[0])).reset_index()
    v_factor.rename(columns={'var': f'var_{factor[i]}'}, inplace=True)
    
    vw_factor = ff_d.groupby('month').agg(weights=('vol', lambda x: 1 / x.iloc[0])).reset_index()
    vw_factor.rename(columns={'weights': f'vw_{factor[i]}'}, inplace=True)
    
    var_m = var_m.merge(v_factor, on='month', how='outer')
    vw_m = vw_m.merge(vw_factor, on='month', how='outer')
```
![image](https://github.com/demihe2004/Quantitative-Asset-Management/assets/135466801/ca7aede0-45f0-4f20-b875-9dc4a27f0cf5)

# 2. Leveraged scaler C

```python
def c(buy_and_hold_r, vw):
    c_factor = buy_and_hold_r.std() / (buy_and_hold_r*vw).std()
    return c_factor

c_list = []
for i in range(len(factor)-2):
    c_list.append(c(ff3_m[f'ann_{factor[i]}'], ff3_m[f'vw_{factor[i]}']))
```

# 3. Get volatility weighted return 
```python
for i in range(len(factor)-2):
    ff3_m.loc[:, f'ann_vw_{factor[i]}'] = ff3_m[f'ann_{factor[i]}'] * ff3_m[f'vw_{factor[i]}'] * c_list[i]
```

# 4. Univariate regression

```python
X = ff3_m["ann_Mkt-RF"] 
y = ff3_m["ann_vw_Mkt-RF"] 

# Add a constant to the X variable
X = sm.add_constant(X)

# Fit the linear regression model
model = sm.OLS(y, X)
results = model.fit()
```
![image](https://github.com/demihe2004/Quantitative-Asset-Management/assets/135466801/249fa065-d77f-409c-a20e-2489a62979f5)

