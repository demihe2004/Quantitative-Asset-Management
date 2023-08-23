# 1. Calculate BE
- merge PRBA dataset with Compustat dataset by GVKEY
``` python
company_merged = company.merge(pension, on = ['gvkey', 'datadate'], how = 'outer')
```

- Check duplicate GVKEY: kept only the latter BE in the same year if there are duplicates
```python
redundant_years = company_merged.groupby('gvkey')['year'].value_counts().loc[lambda x: x == 2]
redundant_groups = redundant_years.index.get_level_values('gvkey').unique()

# Filter the dataframe to include rows with redundant groups
redundant_rows = company_merged[company_merged['gvkey'].isin(redundant_groups)]

max_datadate = redundant_rows.groupby('gvkey')['datadate'].max().reset_index()

# Merge with redundant_rows to keep rows with the later 'datadate' within each redundant group
redundant_rows = redundant_rows.merge(max_datadate, on=['gvkey', 'datadate'])

# Get the non-redundant rows
non_redundant_rows = company_merged[~company_merged['gvkey'].isin(redundant_groups)]

# Concatenate the redundant_rows and non_redundant_rows to get the final result
company_cleaned = pd.concat([redundant_rows, non_redundant_rows], ignore_index=True)
```
- Calculate BE
```python
# create book value and keep only positive BE
company_merged['BE'] = np.where(company_merged['prba'].notna(), company_merged['SHE'] - company_merged['PS'] + company_merged['DT'] - company_merged['prba'], company_merged['SHE'] - company_merged['PS'] + company_merged['DT'])
company_merged['BE'] = np.where(company_merged['BE'] > 0, company_merged['BE'], np.nan)
```
# 2. Calculate ME of June/ December end 
```python
# Add martket equity "me" variable
CRSP_raw['me'] = CRSP_raw['prc'].abs() * CRSP_raw['shrout'] * 1e-3 

# create lagged market cap 
CRSP_raw['lag_me'] = CRSP_raw.groupby('permno')['me'].shift(1)
```
3. Merge BE with ME using linktable
- download from WRDS:
``` python
conn = wrds.Connection(wrds_username=id_wrds)
Linktable = conn.raw_sql("""
                   select gvkey, linkdt, linkenddt, lpermno as permno, linktype, linkprim
                   from crsp.ccmxpf_linktable
                   where substr(linktype,1,1)='L'
                   and (linkprim ='C' or linkprim='P')
                   """)
```
- merge linktable with company, then with CRSP
```python
# merge with company
comp_merge_link = Linktable.merge(company_cleaned, how = 'outer', on = 'gvkey')

# for Multiple gvkey, checked if the date in compustat lands within linkdt and linkenddt in linktable
comp_merge_link['linkdt'] = pd.to_datetime(comp_merge_link['linkdt'])
comp_merge_link['linkenddt'] = pd.to_datetime(comp_merge_link['linkenddt'])
comp_merge_link['datadate'] = pd.to_datetime(comp_merge_link['datadate'])
comp_merge_link = comp_merge_link[comp_merge_link['datadate'].between(comp_merge_link['linkdt'], comp_merge_link['linkenddt'])]

# merge with CRSP
comp_merge_CRSP = comp_merge_link.merge(CRSP_clean, how = 'inner', on = ['permno', 'year'])

# check no na for sign#check if firm have a CRSP stock price for December of year t-1 and June of year t
comp_merge_CRSP['lag_prc'] = comp_merge_CRSP.groupby('permno')['prc'].shift()
comp_merge_CRSP['prc_june'] = np.where(comp_merge_CRSP['month'] == 6, comp_merge_CRSP['prc'], np.nan)
comp_merge_CRSP['prc_june'] = comp_merge_CRSP.groupby(['permno', 'year'])['prc_june'].ffill()
comp_merge_CRSP = comp_merge_CRSP.dropna(subset=['lag_prc', 'prc_june'])
#check if have Compustats data ending in calendar year t-1.
comp_merge_CRSP['lag_be'] = comp_merge_CRSP.groupby('gvkey')['BE'].shift()
comp_merge_CRSP = comp_merge_CRSP.dropna(subset=['lag_be'])
# drop filtering columns and keeping one record for each year
comp_merge_CRSP = comp_merge_CRSP[comp_merge_CRSP['month'] == 6].drop(['lag_prc', 'prc_june',"BE"], axis=1)ls
```
# 3. Calculate B/M ratio 
```python
comp_merge_CRSP['bm'] = comp_merge_CRSP['lag_be'] * 1000 / comp_merge_CRSP['lag_me_dec']
```

# 4. Create bins
```python
# decile bins
def yearly_bins(df,k,s):

    def diff_brkpts(x,k): 
        brkpts_flag = x['brkpts_flag']
        signal = x[s]
        loc_nyse = signal.notna() & brkpts_flag  
        # get all data for nyse listed stocks
        nyse_signal = x.iloc[np.where(loc_nyse)[0]]
        # create breakpoints for each year
        breakpoints = nyse_signal[s].describe(percentiles=[round(1/k * i, 2) for i in range(1, k)])
        breakpoints = breakpoints.loc['min':'max'].values.tolist()
        y = pd.cut(x[s], bins=breakpoints, labels=False) + 1
        return y
    
    bins = df.groupby('year').apply(lambda x: diff_brkpts(x,k))
    bins = bins.to_frame().reset_index(level=0)
    bins.rename(columns={bins.columns[-1]: f'{s}_bin'}, inplace=True)
    
    return bins

# factor bins
def yearly_bins(df,k,s,method):

    def diff_brkpts(x,k,method): 
        brkpts_flag = x['brkpts_flag']
        signal = x[s]
        loc_nyse = signal.notna() & brkpts_flag  
        # get all data for nyse listed stocks
        nyse_signal = x.iloc[np.where(loc_nyse)[0]]
            # create breakpoints for each year
        if method == "bm" or "size":
            breakpoints = nyse_signal[s].describe(percentiles=[round(1/k * i, 2) for i in range(1, k)])
            breakpoints = breakpoints.loc['min':'max'].values.tolist()
            
        if method == "smb":
            breakpoints = nyse_signal[s].describe(percentiles=[0.5])
            breakpoints = breakpoints.loc['min':'max'].values.tolist()
            
        if method == "hml":
            breakpoints = nyse_signal[s].describe(percentiles=[0.3, 0.7])
            breakpoints = breakpoints.loc[['min', "50%","70%",'max']].values.tolist()
        
        y = pd.cut(x[s], bins=breakpoints, labels=False) + 1
        return y
    
    bins = df.groupby('year').apply(lambda x: diff_brkpts(x,k,method))
    bins = bins.to_frame().reset_index(level=0)
    bins.rename(columns={bins.columns[-1]: f'{s}_bin'}, inplace=True)
    
    return bins
```

# Calculate value-weighted return
```python
def wavg(group, avg_name, weight_name):
    d = group[avg_name]
    w = group[weight_name]
    try:
        return (d * w).sum() / w.sum()
    except ZeroDivisionError:
        return np.nan

# compute the value weighted return for size portfolio
size_ret = full_df.groupby(['date','size_bin']).apply(wavg, 'ret','lag_me').to_frame().reset_index()

# compute the value weighted return for bm portfolio
BM_ret = full_df.groupby(['date','bm_bin']).apply(wavg, 'ret','lag_me').to_frame().reset_index()

# compute the value weighted return for factors
# get vwet return for all factors each year/month
strategy = full_df.groupby(['date','smb','hml']).apply(wavg, 'ret','lag_me').to_frame().reset_index().rename(columns={0: 'vwret'})
#get ff fator return for HML and SMB portfolio
ff_ret =strategy.pivot(index='date', columns='factor', values='vwret')
ff_ret['HML_Ret'] = (ff_ret['BH']+ff_ret['SH'])/2-(ff_ret['BL']+ff_ret['SL'])/2
ff_ret['SMB_Ret'] = (ff_ret['BL']+ff_ret['BM']+ff_ret['BH'])/3-(ff_ret['SL']+ff_ret['SM']+ff_ret['SH'])/3
```
# 5. Compute stats for returns
```python
mean = port_ret_size_merged.groupby('bin')['size_log_ex_ret'].mean() * 12
std = port_ret_size_merged.groupby('bin')['size_log_ex_ret'].std() * np.sqrt(12)
SR = mean / std
skew = port_ret_size_merged.groupby('bin')['size_log_ex_ret'].skew()
corr = port_ret_size_merged.groupby('bin')['size_log_ex_ret', 'FF_size_ex_ret'].corr().unstack(level=1)['size_log_ex_ret'].iloc[:, -1].tolist()
```

# 6. Plot returns
```python
#plot 
fig, ax = plt.subplots(figsize=(14, 8))

colors = cm.get_cmap('tab10', 10)
colors = [cm.colors.ColorConverter.to_rgb(color) + (0.7,) for color in colors(range(10))]

for decile in range(1, 11):
    df_decile = port_ret_size_recent[port_ret_size_recent['bin'] == decile]
    color = colors[decile - 1]  # Get color from the colormap based on the decile
    
    ax.plot(df_decile['date'].unique(), df_decile['size_log_ex_ret'].cumsum(), label=f'Bin {decile}', color=color)
    
# Plot the WML
df_wml = port_ret_size_recent[port_ret_size_recent['bin'] == 11]
ax.plot(df_wml['date'].unique(), df_wml['size_log_ex_ret'].cumsum(), label='WML',linestyle='--', color='black', linewidth=1.5)

ax.plot(port_ret_size_recent['date'].unique(), port_ret_size_recent[port_ret_size_recent['bin'] == 1]['SMB_Ret'].cumsum(), label='SMB', linestyle='--', color='grey', linewidth=1.5)

# Set the axis labels and title
ax.set_xlabel('Year')
ax.set_ylabel('Cumulative Excess Return')
ax.legend(fontsize=10, loc='center left', bbox_to_anchor=(1, 0.5))
ax.set_title('Size Portfolio Cumulative Excess Return')

# Display the plot
plt.show()
```
![image](https://github.com/demihe2004/Quantitative-Asset-Management/assets/135466801/6b54b4f4-1d40-4fb9-bdf8-99e2ecb876fa)

![image](https://github.com/demihe2004/Quantitative-Asset-Management/assets/135466801/2b70a61d-1c28-4b20-8b74-4ed46039c5b1)

![image](https://github.com/demihe2004/Quantitative-Asset-Management/assets/135466801/9ec14e18-213e-4d8b-b6ff-6fa9bb3ec4c1)


