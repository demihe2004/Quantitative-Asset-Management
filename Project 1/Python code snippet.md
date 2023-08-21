# 1. Download data from CRSP
```python
conn = wrds.Connection(wrds_username=id_wrds)
dlret_raw = conn.raw_sql("""
                      select a.permno, a.dlstdt, a.dlret, b.ticker
                      from crspq.msedelist as a
                      left join crspq.msenames as b
                      on a.permno=b.permno
                      and b.namedt<=a.dlstdt
                      and a.dlstdt<=b.nameendt
                      """)
conn.close()
conn = wrds.Connection(wrds_username=id_wrds)
CRSP_Stocks = conn.raw_sql("""
                           select a.PERMNO, a.date, a.PRC, a.SHROUT, a.RET, b.SHRCD, b.EXCHCD
                           from crsp.msf as a
                           left join crsp.msenames as b
                           on a.PERMNO = b.PERMNO
                           and b.namedt <= a.date
                           and a.date <= b.nameendt
                           """)
conn.close()
CRSP_Stocks.to_pickle(data_folder + 'CRSP_Stocks.pkl') 
CRSP_Stocks
```
# 2. Get EW return with lagged market cap
```python
CRSP_raw['month_number'] = CRSP_raw['date'].dt.month + (CRSP_raw['date'].dt.year-1926)*12
CRSP_raw["year"] = CRSP_raw['date'].dt.year
CRSP_raw["month"] = CRSP_raw['date'].dt.month

for m in range(2,1165):
    r_m_prior = CRSP_raw[CRSP_raw['month_number']==(m-1)]
    r_m_prior["wt"] = r_m_prior.me/ r_m_prior.me.sum()
    r_m = CRSP_raw[CRSP_raw['month_number']== m]
    r_m = r_m.merge(r_m_prior[["permno", "wt"]], on="permno", how="left")
    r_m = r_m[r_m['wt'].notna()].copy()
    Monthly_CRSP_Stocks['Stock_lag_MV'].iloc[(m-1)] = sum(r_m_prior["me"])
    Monthly_CRSP_Stocks['Stock_Ew_Ret_MV'].iloc[(m-1)] = r_m["ret"].mean()
    Monthly_CRSP_Stocks['Stock_Vw_Ret_MV'].iloc[(m-1)] = sum(r_m["ret"]*r_m["wt"])
```
# 3. Compute stats of FF time series 

![image](https://github.com/demihe2004/Quantitative-Asset-Management/assets/135466801/769708af-cbc5-41ac-b6a7-ffe000751ca9)

# 4. Compare replicate quality

![image](https://github.com/demihe2004/Quantitative-Asset-Management/assets/135466801/48e53521-0c41-4159-9545-60049b26578b)



