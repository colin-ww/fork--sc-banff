# Donor Imputation

* Execution: *banff.donorimp()*
* SDE function types: *Treatment*
* Input status flags: *FTI (required), FTE(optional), I-- (optional)*
* Output status flags: *IDN*

## Description

Performs nearest neighbour donor imputation such that each imputed record satisfies the specified post-imputation edits.

The donorimp procedure splits records into recipients (records requiring imputation) and donors (records not requiring imputation that pass the edits). For each recipient, the procedure performs the following steps:

1. From the fields in the edits, a subset are chosen as matching fields for the purpose of distance calculations. This selection can vary depending on which fields require imputation. Users can also specify must-match fields (`must_match`), which will automatically be included in distance calculations alongside the system-selected ones.
2. Matching fields are transformed into normalized ranks to remove the effect of scale and clustering from the data. Without this transformation, original data with wide ranges, such as dollar values, would always dominate the distance calculation.
3. Distances between the recipient and donors are calculated using a L-$\infty$ norm on the transformed matching fields. This is sometimes referred to as the minimax distance because the closest donor is the one with the smallest maximum absolute difference between the transformed values of its matching fields and those of the recipient.
4. From the donors, a search algorithm is used to efficiently find the closest donor whose values allow the recipient record to pass the user-specified post-imputation edits (`post_edits`). These are typically a more relaxed form of the edits to ensure a donor can be found.

*Note: The Banff distance metric will usually select different donors than a typical Euclidean distance metric. This is by design. When using Euclidean distance metrics, scale differences and skewed distributions in economic data typically result in a distance metric that is dominated by a single field such as revenue. The Banff distance metric ensures that all matching fields are given the same weight in the distance calculation.*

Recipients are defined as any record with at least one field within the edits requiring imputation, as indicated by an FTI (Field to Impute) flag on the input status (`instatus`) file. Donors are defined as any record satisfying all the edits that is not a recipient. A record that is neither is called a *mixed* record and is dropped before processing: it has no field to impute within the edits (so it is not a recipient), but it has an FTI flag on one of the `must_match` variables outside the edits (so it cannot be a donor). The donorimp procedure requires a set of edits; for a version of donor imputation that does not, please see the `massimp` procedure.

There are a number of ways to exclude records or values from the donor pool. Records can be excluded using the  `exclude_where_indata` or `data_excl_var` parameters. This does not exclude them from the procedure completely; they may still be included as recipients if they require imputation. Records that have previously been imputed can also be excluded from the donor pool using the `eligdon` (eligible donor) parameter. The parameter `n_limit` will limit the number of times a single donor is used for imputation. Users may sometimes identify values that do not require imputation, but are sufficiently unusual that they should not be donated to other records; these should be flagged as FTE (Field to Exclude) on the instatus file.

The Banff distance metric does not accommodate categorical variables. Instead, users may create by-groups by specifying `by` variables. These by-groups act as imputation classes. Use the `min_donors` and `percent_donors` parameters to ensure an appropriate number or ratio of recipients and donors exist in each imputation class before performing imputation.

For a full mathematical description of the procedure methods, with examples, please see the [Functional Description](../Banff%20Functional%20Description.pdf).

## Input and output tables

Descriptions of input and output tables are given below. Banff supports a number of input and output formats; please see the Banff User Guide for more information.

| Input table   | Description |
| ------------- | ----------- |
| indata        | Input statistical data. Mandatory. |
| instatus      | Input status file containing FTI, FTE and I-- status flags. Mandatory.<br><br> The instatus file is required by donorimp to identify recipients, as records with an FTI flag in at least one of the fields within `edits`. Fields flagged as FTE are not imputed but are not used as values for imputation. If `eligdon = "ORIGINAL"`, then the procedure will also look for imputation flags of the form I--. |

| Output table       | Description |
| -----------------  | ------------------------------------------------- |
| outdata            | Output statistical table containing imputed data. <br><br> Note that outdata will only contain successfully imputed records and affected fields. Users should update indata with the values from outdata before continuing the data editing process. |
| outstatus          | Output status file identifying imputed fields with IDN status flags, and their values after imputation. |
| outdonormap        | Output table of recipient-donor pairs for successfully imputed records. |
| outmatching_fields | Output table of matching fields for each recipient.<br><br> Matching fields are used by the donor imputation procedure to calculate distances between recipients and potential donors. Matching fields can take on one of three statuses: MFS (system selected), MFU (user-specified) or MFB (both system and user-specified). |

For details on the content of output tables, please see the [Output Tables](../output_tables.md) document.

## Parameters

| Parameter           | Python type  | Description | 
| ------------------- | -----------  | ----------- |
| unit_id             | str          | Identify key variable (unit identifier) on indata. Mandatory. <br><br> Must be unique for each record. Records with a missing value are dropped before processing. |
| edits               | str          | List of consistency edits. Mandatory. <br><br> Example: `"Revenue - Expenses = Profit; Revenue >= 0; Expenses >= 0;"` |
| post_edits          | str          | List of edits a recipient must satisfy after imputation. <br><br> If not specified, `edits` are used as the post-imputation edits. |
| min_donors          | int          | Minimum number of donors needed in a by-group to perform imputation. Default=30. |
| percent_donors      | float        | Minimum percentage of donors needed in a by-group to perform imputation (at least 1 and less than 100). Default=30. |
| n                   | int          | Maximum number of donors to try for a given recipient. Mandatory. |
| eligdon             | str          | Identify eligible donors as 'ANY' (include all potential donors) or 'ORIGINAL' (exclude records with imputed values). Default='ORIGINAL'. <br><br> Not case sensitive; the first letter alone ('A' or 'O') is also accepted. Any other value results in an error. |
| n_limit             | int          | Limit the number of times a donor can be used. <br><br> See [Limiting the repeated use of donors](#limiting-the-repeated-use-of-donors). |
| random              | bool         | Randomly select donors for recipients without matching fields. |
| rand_num_var        | str          | Identify random number variable on indata. <br><br> The variable must be numeric, with non-negative values less than or equal to 1, and should be unique for each record. |
| seed                | int          | Specify a seed for the random number generator.<br><br> The seed is used to ensure consistent results from one run to the next. If not specified or specified as a non-positive value, a random number is generated by the procedure. Must be a whole number; a fractional value results in an error. |
| mrl                 | float        | Multiplier ratio limit used to limit number of times a donor can be used. <br><br> See [Limiting the repeated use of donors](#limiting-the-repeated-use-of-donors). |
| must_match          | str          | List of variables on indata to be used in distance calculations. |
| data_excl_var       | str          | Identify donor exclusion variable on indata.<br><br> The variable must be a string (character) column. A record is excluded from the pool of donors when the value of this variable is exactly the uppercase letter `'E'`. Any other value (including lowercase `'e'`, `'1'`, `'Y'` or a blank) is **not** an exclusion, and no warning is given. Recipients are never excluded, whatever the value of this variable.<br><br> `data_excl_var` cannot be combined with `exclude_where_indata`. |
| exclude_where_indata | str         | Donor exclusion expression using SQL syntax.<br><br> A new column will be temporarily added and matching rows satisfying the SQL expression will be flagged for exclusion from the pool of donors. Then, the column will be used as the `data_excl_var` parameter. `exclude_where_indata` cannot be combined with `data_excl_var`.<br><br> Imputation will still be carried out on fields that are FTI even if they satisfy the expression in `exclude_where_indata`. |
| accept_negative     | bool         | Treat negative values as valid. Default=False. <br><br> By default, a positivity edit is added for every variable in the list of edits; this parameter permits users to remove this restriction. If required, users may directly add positivity edits for individual variables. |
| by                  | str          | Variable(s) used to partition indata into by-groups for independent processing. <br><br> In donorimp, by-groups can also be seen as imputation classes. <br><br> Example: `by = "province industry"`. |
| prefill_by_vars     | bool         | Adds by-group variable(s) to input status file(s) to improve performance. Default=True. |
| presort             | bool         | Sorts input tables before processing, according to procedure requirements. Default=True. |
| no_by_stats         | bool         | Reduces log output by suppressing by-group specific messages. Default=False. |
| display_level       | int          | Value (0 or 1) to request detail output to the log in relation to the random number variable. Default=0. |

## Notes

### Multiple equivalent solutions

In some cases, for a given recipient, there may be multiple equidistant donors (i.e. having the same distance from the recipient) whose values would allow the recipient to pass the edits. When this occurs, the procedure selects one of these solutions at random.

For development or testing purposes, users may wish to produce consistent results over multiple runs of the procedure, and may do so using the `seed` or `rand_num_var` parameters. Both ensure that the same solutions will be selected from one run to the next, if executed on the same set of inputs. Note that if both `seed` and `rand_num_var` are specified, `seed` is ignored. If neither is specified, the system generates a default seed.

### Limiting the repeated use of donors

Users may limit the repeated use of donors with the interrelated `n_limit` and `mrl` parameters. The donor limit is calculated as follows, depending on whether one or both are specified:

| `n_limit` | `mrl`  | Donor Limit                                        | 
| --------- | -------| ------------------------------------------------   |
| No        | No     | Number of times a donor can be used is unlimited.  |
| No        | Yes    | $\lceil mrl \cdot \lceil recipients/donors \rceil \rceil$.   |
| Yes       | No     | `n_limit`.                                         |
| Yes       | Yes    | $\max(n\_limit, \lceil mrl \cdot \lceil recipients/donors \rceil \rceil)$.   |

where $\lceil \cdot \rceil$ means rounded **up** to the next whole number, and *recipients* and *donors* are the counts within the by-group. For example, with 25 recipients, 10 donors and `mrl=1.2`, the donor limit is $\lceil 1.2 \cdot \lceil 25/10 \rceil \rceil = \lceil 1.2 \cdot 3 \rceil = 4$. The calculated limit is shown as `DONORLIMIT` in the `outdonormap` table and in the log.

A donor that has reached the donor limit is retired from the donor pool and is not counted toward the maximum number of donors to try (`n`). Imputation stops for a by-group once all donors have reached the limit, so some recipients may be left unimputed; using `mrl` of 1 or more ensures at least one donor is available for each recipient.

When limiting the number of donors with the `n_limit` parameter, the number of remaining donors may end up being less than `min_donors`. In such a case, mass imputation will continue and ignore `min_donors` which was validated at the beginning. The same applies for `percent_donors`.