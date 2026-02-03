## Index

- [Traditional DCF Model](#Traditional-DCF-Model)
- [SOTP Hybrid Model](#SOTP-Hybrid-Model)


## Traditional DCF Model


<img src="https://github.com/fabiobassini/financial_valuation_models/SOFI/imgs/dcf/1.png" alt="overview" width="400"/>
<img src="https://github.com/fabiobassini/financial_valuation_models/SOFI/imgs/dcf/2.png" alt="overview" width="400"/>
<img src="https://github.com/fabiobassini/financial_valuation_models/SOFI/imgs/dcf/3.png" alt="overview" width="400"/>

### Detailed Functionality
The DCF model forecasts financials over an explicit period (default: 10 years) with three phases:
- **High-Growth Phase** (default: 4 years): Revenue grows at a high rate (e.g., 29.6%), with EBITDA margin ramping from current to target.
- **Transition Phase** (remaining explicit years): Growth fades linearly to transition rate.
- **Terminal Phase**: Perpetual growth at terminal rate, or via exit multiple on EBITDA.

**Key Calculations**:
1. **Revenue Projection**: Starting from base revenue, apply phased growth:  
   `new_rev = previous_rev * (1 + growth_rate)`  
   Growth rate depends on phase (high, transition via linear interpolation).

2. **EBITDA**: `ebitda = new_rev * margin`  
   Margin ramps linearly: `margin = current_margin + (target_margin - current_margin) * progress` (progress = min(year / high_growth_years, 1)).

3. **NOPAT**: `nopat = ebitda * (1 - tax_rate)`  
   Tax rate ramps from near-term to long-term over 10 years.

4. **Free Cash Flow (FCF)**: `fcf = nopat + (new_rev * da_rate) - (new_rev * capex_rate) - (delta_rev * wc_rate)`  
   - Depreciation & Amortization (D&A): % of revenue.
   - Capex: % of revenue.
   - Working Capital (WC): % of revenue change.

5. **Discounting**: Discount FCF at WACC: `discounted_fcf = fcf / (1 + wacc)^year`.

6. **Terminal Value**: `terminal_fcf = last_fcf * (1 + terminal_growth)`  
   `terminal_value = terminal_fcf / (wacc - terminal_growth)` (Gordon Growth, floored denominator at 0.005).  
   Alternatively, uses terminal multiple on EBITDA. Discount to PV.

7. **Enterprise Value (EV)**: Sum of PV FCF + PV terminal.  
   **Equity Value**: EV - net debt (negative if net cash).  
   **Price per Share**: Equity Value / shares (adjusted for dilution).

8. **Additional Features**:
   - **Dilution**: Shares grow at dilution rates during phases.
   - **Sensitivity**: Matrix of price vs ±5% growth / ±2% WACC.
   - **Reverse DCF**: Solves for implied high-growth rate to match current price (using Brentq optimizer).
   - **Price Trajectory**: Projects future prices via fading EV/EBITDA multiples.
   - **Peers**: Compares implied fwd EV/EBITDA.

**Scenarios**: Pre-sets like "base" (guidance-aligned), "bull", "bear" adjust key params.

### Parameters
Parameters are stored in `default_params`. All values from 8-K (Q4 25) unless noted.

| Parameter | Default Value | Description | Calculation/Source/Reference |
|-----------|---------------|-------------|------------------------------|
| `revenue_base` | 3.613354e9 | 2025 base revenue for projections. | Direct: "Total net revenue" 2025 ($3,613,354,000) from Consolidated Results Summary table. |
| `current_ebitda_margin` | 0.2935 | Starting EBITDA margin. | Adjusted EBITDA ($1,053,898,000) / Adjusted net revenue ($3,591,411,000) from Consolidated Results Summary. |
| `target_ebitda_margin` | 0.34 | Target EBITDA margin. | Implied from 2026 guidance (text: "Adjusted EBITDA up 60% to a record $318 million" Q4, extrapolated to year). |
| `high_growth_rate` | 0.296 | High-growth revenue rate. | Implied 2026 adj. revenue ~$4.655B (29.6% from 2025 adj. revenue $3,591M; text guidance). |
| `high_growth_years` | 6 | Years of high growth. | Assumption: Conservative post-guidance period. |
| `transition_growth_rate` | 0.10 | Transition growth rate. | Assumption: Fade to terminal. |
| `terminal_growth` | 0.035 | Perpetual growth rate. | Assumption: GDP + inflation. |
| `capex_rate` | 0.045 | Capex % revenue. | Estimated from noninterest expenses (tech capex implied). |
| `wc_rate` | 0.08 | WC % delta revenue. | Assumption: Efficiency improvements. |
| `tax_near` | 0.085 | Near-term tax rate. | Effective tax implied from net income (text). |
| `tax_long` | 0.21 | Long-term tax rate. | US statutory rate. |
| `wacc` | 0.158 | WACC (≈ CoE). | rf (0.043) + beta (2.10) * erp (0.055); net cash position. |
| `shares` | 1.270569e9 | Diluted shares. | Estimated from EPS and capital ratios (Table 9 CET1). |
| `explicit_years` | 10 | Explicit forecast years. | Standard DCF horizon. |
| `terminal_multiple` | 22.0 | Terminal EV/EBITDA. | Assumption: Fintech peers avg. |
| `net_debt` | -3.541 | Net debt (negative = cash). | Cash + restricted ~$5.356B - debt ~$1.815B (implied balance sheet). |
| `dilution_rate_high` | 0.035 | High-growth dilution rate. | Assumption: Share issuance. |
| `dilution_rate_transition` | 0.005 | Transition dilution rate. | Assumption: Reduced issuance. |
| `da_rate` | 0.065 | D&A % revenue. | Estimated from expenses (~6.5%). |
| `rf` | 0.043 | Risk-free rate. | Current 10Y Treasury (external ref). |
| `beta` | 2.10 | Equity beta. | High volatility post-2025. |
| `erp` | 0.055 | Equity risk premium. | Standard. |
| `current_price` | 22.20 | Current price. | Indicative market price (Feb 3, 2026). |


## SOTP Hybrid Model

<img src="https://github.com/fabiobassini/financial_valuation_models/SOFI/imgs/sotp/4.png" alt="overview" width="400"/>
<img src="https://github.com/fabiobassini/financial_valuation_models/SOFI/imgs/sotp/5.png" alt="overview" width="400"/>
<img src="https://github.com/fabiobassini/financial_valuation_models/SOFI/imgs/sotp/6.png" alt="overview" width="400"/>
<img src="https://github.com/fabiobassini/financial_valuation_models/SOFI/imgs/sotp/7.png" alt="overview" width="400"/>

### Detailed Functionality
SOTP values segments independently, summing to total equity value minus corporate drag.

1. **Lending Segment (Residual Income Model)**:  
   - Loop over 10 years:  
     `net_interest_income = current_assets * nim`  
     `non_interest_income = current_assets * non_interest_rate`  
     `total_revenue = net_interest_income + non_interest_income`  
     `provision_losses = current_assets * provision_rate`  
     `non_interest_expense = total_revenue * efficiency_ratio`  
     `net_income_pre_tax = total_revenue - provision_losses - non_interest_expense`  
     `net_income = net_income_pre_tax * (1 - tax_rate)`  
     `residual_income = net_income - (current_tbv * lending_coe)`  
     Discount RI at lending_coe: `pv_ri = residual_income / (1 + lending_coe)^year`  
   - Update assets/TBV: `* (1 + asset_growth)` each year.  
   - Terminal RI: Last RI * (1 + terminal_growth) / (lending_coe - terminal_growth), discounted.  
   - Total Bank Value: lending_tbv + sum(pv_ri) + pv_terminal_ri.

2. **Tech/Financial Services Segment (Mini-DCF)**:  
   - Project revenues with high/transition growth (similar to DCF).  
   - EBITDA = revenue * target_margin.  
   - NOPAT = EBITDA * (1 - tax_rate).  
   - FCF = NOPAT + (revenue * da_rate) - (revenue * capex_rate) - (delta_rev * wc_rate).  
   - Discount FCF at cost_of_equity.  
   - Terminal: Last EBITDA * (1 + terminal_growth) * exit_multiple, discounted.  
   - Total Tech Value: Sum PV FCF + PV terminal.

3. **Corporate Drag**: abs(corporate_costs) * corporate_cost_multiple (capitalized perpetuity).

4. **Total SOTP**: Bank + Tech - Drag, divided by shares for per-share value.

### Parameters
Parameters in `default_hybrid_params`. Derived from 8-K segments/loans.

| Parameter | Default Value | Description | Calculation/Source/Reference |
|-----------|---------------|-------------|------------------------------|
| `shares` | 1.270569e9 | Diluted shares. | Same as DCF. |
| `cost_of_equity` | 0.158 | CoE for tech. | Same as DCF WACC. |
| `current_price` | 22.20 | Current price. | Indicative. |
| `tax_rate` | 0.085 | Tax rate. | Effective from 8-K. |
| `lending_earning_assets` | 38.037e9 | Lending assets. | Total loans ~$32.44B + adjustments (Table 8). |
| `lending_tbv` | 8.863e9 | Lending TBV. | CET1 proxy (Table 9). |
| `lending_nim` | 0.0572 | NIM. | Explicit (text). |
| `lending_provision_rate` | 0.008 | Provisions % assets. | Adjusted from year provisions / avg loans (text). |
| `lending_efficiency_ratio` | 0.35 | Expenses % revenue. | From lending contribution (text). |
| `lending_asset_growth` | 0.15 | Asset growth. | YoY loans ~38%, conservative for future. |
| `lending_terminal_growth` | 0.04 | Terminal growth. | Assumption. |
| `lending_non_interest_income_rate` | 0.008 | Non-interest % assets. | Q4 $54M / assets, grown for guidance. |
| `lending_coe` | 0.075 | Lending CoE. | Assumption (lowered for bank stability). |
| `tech_fs_revenue_base` | 1.992e9 | Tech/FS revenue. | FS + Tech segments (implied). |
| `tech_fs_high_growth_rate` | 0.32 | High growth. | Assumption (high-growth segment). |
| `tech_fs_high_growth_years` | 5 | High-growth years. | Assumption. |
| `tech_fs_transition_growth_rate` | 0.12 | Transition growth. | Assumption. |
| `tech_fs_explicit_years` | 10 | Explicit years. | Standard. |
| `tech_fs_target_ebitda_margin` | 0.38 | Target margin. | Assumption (higher scalability). |
| `tech_fs_capex_rate` | 0.035 | Capex % revenue. | Assumption. |
| `tech_fs_wc_rate` | 0.06 | WC % delta revenue. | Assumption. |
| `tech_fs_da_rate` | 0.03 | D&A % revenue. | Assumption. |
| `tech_fs_exit_multiple` | 24.0 | Terminal multiple. | Assumption (tech peers). |
| `tech_fs_terminal_growth` | 0.04 | Terminal growth. | Assumption. |
| `corporate_costs` | -1.222e9 | Annual costs. | Unallocated noninterest (total expenses - segments). |
| `corporate_cost_multiple` | 9.0 | Capitalization multiple. | Assumption (~11% discount perpetuity). |
