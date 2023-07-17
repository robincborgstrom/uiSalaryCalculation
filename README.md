# Revised salary calculation for KHO 2023:42

## Overview
1. Starting amount
  - Invoice amount without VAT, allowances without VAT and expenses with VAT
2. Service fee + Quick pay fee
  - 2.8% and 2.5% of starting amount
3. Social contribution
  - invoice items VAT 0 - service fee amount + quick pay fee amount
4. Gross salary
  - Starting amount - (service fee amount + quick pay fee amount) - social contribution
5. Net Salary
  - Gross salary - personal tax
6. Paid amount
  - net salary + expense + allowance - yelAccidental - normal_gov_pay - zero_gross_service_fee (if the gross is negative. See Conditions)

## Database
sp_GetInvoiceInfoWages1 provides the values
## Salary Calculation in salary.reducer SELECT_ROW_SALARY_SUCCESS
- action.result.taxResult
- salaryTaxPercent
- state.selectedRows
- state.newSalary

## Key arrays
- inv_sum: ```state.newSalary[el].sumInvWithTax - state.newSalary[el].sumInvVat - state.newSalary[el].suminvAllowVat```
  - sumInvWithTax: sumWithoutTax ```(SUM(items.sum_tax_free)) + sumVat (SUM(items.vat)) // items: invoice_items```
  - sumInvVat: sumVat ```(SUM(items.vat)) // items: invoice_items```
  - suminvAllowVat: ```SUM(items.vat) WHERE items.invoice_allowance_id <> -1 // selects only allowances```

inv_sum will contain: **invoice items VAT 0 + allowance VAT 0 + expense with VAT**

- inst_pay_final: checks if invoice is marked with quick pay
  - If it's a quick pay invoice: ```inst_pay_final.push(quick_pay_percentage)```
- ser_per_final: copy of inst_pay_final
- ser_cost: uses inv_sum to calculate service fee + quick pay fee. Changes if it's a discount salary.
  - ```inv_sum[i] * ser_per_final[i] * 0.01 * 1.24 + inv_sum[i] * service_percentage * 0.01```

// New
- payRollStartAmount: selects only invoice items vat 0<br>
Used to check discount, setting newSalarySummary.sumInvWithoutTax and setting newSalarySummary.palkka. Since previously discount<br>
was checked by gross amount which was invoice items vat 0.
- social_tax: calculates new social contribution per KHO 2023:42 // Could be refactored see 'Reasoning *'

- palkka: calculates the gross salary. Subracting invoice items VAT, allowance VAT, expenses and allowances from total amount with VAT.

- **Rest of the arrays are the same**

## Conditions
'if (gross_sum <= 0)': Since the gross sum and social contribution cannot be 0, setting it manually to 0 here. There's a case when<br>
there are invoice items, expenses or allowances, where the invoice items are lower than the service fee. That means that we're<br>
subtracting the invoice items to get the negative gross and then setting it to 0. Then at the end of the calculation we're <br>
summing expenses and allowances and subtracting the remaining service fee. <br>
Using variable 'isAllowanceExpenseServiceCostHigherThanInvoiceItems' to check for this case.<br>
'isAllowanceExpenseServiceCostHigherThanInvoiceItems': Checks for invoice items vat  0 < service_fee 
'zero_gross_service_fee': Used in the final 'paid_sum' to subtract full service fee in case it's only expense or allowances and if <br>
it is invoice items VAT 0 < service fee then it  recalculates the service fee with subtracting invoice items.<br>

**Rest of the conditions are the same**

## Reasoning
Moved discount check and calculation to before service fee calculation to ignore service fee if there's a discount.<br>
Moved yel_option check before social_contribution calculation since it wil be ignored if there is 'no_yel'.<br>
Moved expense to before social contribution calculation since we're subtracting the expense from the total amount.<br>
Moved allowance to before social contribution calculation since we're subtracting the allowance from the total amount.<br>
\* The above could be refactored to use 'sumInvWithoutTax'<br>

**Rest of the logic is the same**