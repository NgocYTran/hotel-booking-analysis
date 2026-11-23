# Hotel Booking Analysis

## 1. Project Overview

This project looks at hotel booking data for a city hotel and a resort hotel. I want to understand booking behavior, cancellations, and room pricing. The planned business questions are:

- How do bookings and cancellations vary by hotel, season, and booking channel?
- What factors are related to a booking being canceled?
- How does the average daily rate (ADR) change across hotels, room types, and months?
- Which market segments bring in the most estimated room revenue?

So far, I have finished the data quality assessment, cleaning, and preparation stage. The exploratory analysis, Tableau dashboard, and business recommendations are still in progress.

## 2. Dataset Overview

- **Source:** [Hotel Booking dataset on Kaggle](https://www.kaggle.com/datasets/mojtaba142/hotel-booking?select=hotel_booking.csv), uploaded by Mojtaba
- **Original data:** According to the Kaggle description, the data comes from the article *Hotel Booking Demand Datasets* by Nuno Antonio, Ana Almeida, and Luis Nunes (*Data in Brief*, Volume 22, February 2019).
- **License (as listed on Kaggle):** Data files © Original Authors
- **Size:** 119,390 rows and 36 columns
- **Time period:** Scheduled arrival dates from July 1, 2015 to August 31, 2017
- **Hotels:** City Hotel (79,330 bookings) and Resort Hotel (40,060 bookings)

Each row is one booking. The columns include:

| Category | Example columns |
|---|---|
| Booking details | `hotel`, `lead_time`, `arrival_date_year`, `arrival_date_month`, `arrival_date_day_of_month` |
| Stay and guests | `stays_in_weekend_nights`, `stays_in_week_nights`, `adults`, `children`, `babies` |
| Sales channel | `market_segment`, `distribution_channel`, `agent`, `company`, `customer_type` |
| Room and price | `reserved_room_type`, `assigned_room_type`, `adr`, `meal`, `deposit_type` |
| Booking outcome | `is_canceled`, `reservation_status`, `reservation_status_date` |
| Customer information | `name`, `email`, `phone-number`, `credit_card` |

The Kaggle description says the four customer information columns were artificially created and added to the original dataset.

## 3. Tools and Technologies

- **Python**: data cleaning and preparation
- **Pandas**: data inspection, transformation, and validation
- **Jupyter Notebook**: documenting the workflow step by step
- **Tableau** (planned): dashboard and visualization

## 4. Data Quality Assessment and Cleaning

### Missing Values

Four columns had missing values. I looked at each one before deciding how to handle it.

| Column | Missing | What I found | Decision |
|---|---:|---|---|
| `children` | 4 | All 4 bookings had 2–3 adults and no babies. They make up only 0.0034% of the dataset. | Filled with 0. With so few rows involved, this choice has very little effect on the analysis. |
| `country` | 488 | The missing values were spread across both hotels and several market segments. The real country can't be determined. | Filled with `"Unknown"` so these bookings stay in the analysis without guessing a country. |
| `agent` | 16,340 | Most were Direct (7,625) or Corporate (5,543) bookings, suggesting some bookings don't involve a travel agent. However, 3,167 came through the TA/TO channel, including 165 marked as Direct in `market_segment`. | Left as missing because the agent can't be reliably inferred (see below). |
| `company` | 112,593 | 94.31% of bookings had no company ID. Missingness depends on the market segment. Most Corporate and Aviation bookings have a company ID, while most Direct and Online TA bookings don't. | Left as missing. Company information seems to be recorded mainly for business-related bookings, but I couldn't confirm what a missing value means. |

I checked whether one column could fill in the other. I decided it couldn't, for two reasons:

1. **They represent different roles.** `agent` identifies a travel agency, and `company` identifies an associated company, so they can't be treated as the same entity.
2. **The relationship is many-to-many.** Agent 9 is linked to 9 different companies, and company 281 is linked to 4 different agents. Only 217 bookings have both an agent and a company ID.

### Duplicate Records

The raw dataset contained no duplicate rows, so each row represents a separate booking
(see [Final Dataset Validation](#6-final-dataset-validation)).

### Zero-Night and Zero-Guest Bookings

**Zero-night bookings:** 715 bookings had 0 weekend nights and 0 weeknights.

| Reservation status | Bookings |
|---|---:|
| Check-Out | 680 |
| Canceled | 22 |
| No-Show | 13 |

I first thought the 680 checked-out zero-night bookings were complimentary stays because all of them had an ADR of 0. However, only 14 were in the Complementary segment, so that idea didn't hold up. All 680 had a reservation status date that matched the scheduled arrival date, which suggests they could be same-day reservations. There wasn't enough evidence to treat them as errors, so I kept them. They are left out of the estimated revenue calculation, and I'll handle them separately in any analysis of overnight stays.

**Zero-guest bookings:** 180 bookings had 0 adults, 0 children, and 0 babies. A canceled or no-show booking could be missing guest details, but a completed stay with zero guests looks like a data entry error or an administrative booking. 31 of these were checked out with an ADR above 0. The actual guest counts can't be determined, so I didn't fill in any values. I kept the rows and will leave them out of guest-related analysis where needed.

### Negative and Zero ADR

- **Negative ADR:** Only 1 booking had a negative ADR (-6.38, Check-Out). A negative room rate doesn't make sense, so I kept the row but left it out of the estimated revenue calculation.
- **Zero ADR:** 1,959 bookings had an ADR of 0. I focused on the **1,066 checked-out bookings with at least one night**. 574 of them (53.8%) were in the Complementary segment, which supports the idea that these were genuinely free stays. The other **492** couldn't be explained, but there wasn't enough evidence to call them errors. I kept them and flagged them.

### ADR Outliers (IQR Method)

ADR ranged from -6.38 to 5,400, with a mean of 101.83 and a median of 94.58. I used the interquartile range (IQR) method to find outliers:

| Measure | Value |
|---|---:|
| Q1 | 69.29 |
| Q3 | 126.00 |
| Lower bound (Q1 − 1.5 × IQR) | about -15.78 |
| Upper bound (Q3 + 1.5 × IQR) | about 211.07 |
| Bookings outside the bounds | 3,793 |
| Mean ADR without outliers | 97.04 |

Because the lower bound is below zero, all 3,793 outliers are high values. Their reservation statuses were 2,324 Check-Out, 1,446 Canceled, and 23 No-Show.

- The highest ADR (5,400) was a canceled City Hotel booking for room type A. It is far above every other value, with the next highest being 510.
- Many of the other high-ADR checked-out bookings were Resort Hotel stays in July and August in room types G and H. That fits with peak-season pricing and larger rooms.

High prices can be real, so I kept these rows and created a flag column for them. That way I can include or exclude them later depending on the analysis.

### Reservation Status and Date Consistency

- **Cancellation fields:** `is_canceled` matched `reservation_status` in every row. All Canceled (43,017) and No-Show (1,207) bookings have `is_canceled = 1`, and all Check-Out bookings (75,166) have `is_canceled = 0`.
- **Status date before arrival date:** 42,137 bookings had a reservation status date earlier than the scheduled arrival date. 42,136 of these were cancellations, which is expected because guests usually cancel before arriving. Only **1 was a No-Show**, which doesn't make sense, so I flagged it.

### Categorical Value Validation

I checked the value counts for `hotel`, `distribution_channel`, `market_segment`, `customer_type`, `meal`, `reserved_room_type`, `assigned_room_type`, and `deposit_type`. I didn't find spelling variations or inconsistent labels. A few columns contain an `Undefined` category: `meal` (1,169), `distribution_channel` (5), and `market_segment` (2). I kept these values as they are.

### Removing Customer Information

I dropped `name`, `email`, `phone-number`, and `credit_card`. These columns aren't needed for the analysis, and personal or payment details shouldn't be shared in a public repository, even though the Kaggle description says they were generated.

## 5. Derived Columns and Data Preparation

| Column | How it was created | Purpose |
|---|---|---|
| `booking_id` | Row number from 1 to 119,390, added after cleaning. No rows were removed or reordered, so it follows the original file order. | Gives each booking its own ID in the cleaned file |
| `scheduled_arrival_date` | Combined arrival year, month, and day into one date | Makes time-based analysis and date comparisons easier |
| `total_guests` | `adults + children + babies` | Finds zero-guest bookings and measures group size |
| `total_nights` | `stays_in_weekend_nights + stays_in_week_nights` | Measures length of stay and finds zero-night bookings |
| `date_difference` | Days between `reservation_status_date` and `scheduled_arrival_date` | Checks whether status dates make sense for each reservation status |
| `estimated_room_revenue` | `adr × total_nights` | Gives a rough revenue estimate for completed stays |

**Why I added `booking_id`:** The source data has no booking ID. My duplicate check at the start found no duplicate rows, which confirmed that each row is a separate booking. After cleaning, I added `booking_id` so each booking can be identified and counted in later analysis and in Tableau.

`estimated_room_revenue` is only calculated for bookings that are **checked out**, have **at least one night**, and have an **ADR of 0 or higher**. All other rows are left blank. Canceled bookings and No-Shows didn't produce a completed stay, and zero-night or negative-ADR bookings would give misleading values. This leaves **74,485** bookings with an estimated revenue value. The median is 260.64 and the maximum is 7,590.

### Data Quality Flags

Instead of deleting questionable rows, I created three True/False flags. This keeps the full dataset, and in Tableau I can choose whether to include these rows.

| Flag | Rule | Rows flagged |
|---|---|---:|
| `adr_outlier_flag` | ADR is outside the IQR bounds | 3,793 |
| `unexplained_zero_adr_flag` | ADR = 0, checked out, at least one night, and not Complementary | 492 |
| `date_inconsistency_flag` | Status date is before the arrival date and the booking was not canceled | 1 |

## 6. Final Dataset Validation

- **Final shape:** 119,390 rows and 41 columns. No rows were removed. 4 customer columns were dropped, and 9 new columns were added.
- **Remaining missing values (intentional):**

| Column | Missing | Reason |
|---|---:|---|
| `agent` | 16,340 | No agent recorded for the booking |
| `company` | 112,593 | No company recorded for the booking |
| `estimated_room_revenue` | 44,905 | Only calculated for checked-out bookings with at least one night and ADR ≥ 0 |

- **Data types:** `reservation_status_date` and `scheduled_arrival_date` were converted to dates, and the three flags are stored as True/False.
- **Export:** The cleaned data was saved as `dataset/hotel_booking_cleaned.csv`. I read the file back in and confirmed it has the same shape (119,390 × 41) and column names as the cleaned DataFrame.

## 7. Data Limitations

- **No original booking ID.** The source data has no booking ID. booking_id is a row number I added, so it can't be matched to the hotels' own records.
- **Estimated room revenue is not actual revenue.** It is ADR multiplied by the number of nights. It doesn't include meals, extra charges, discounts, taxes, or cancellation fees, and it assumes the ADR applied to every night of the stay.
- **Unusual records aren't necessarily wrong.** High ADR values, zero-ADR stays, zero-night stays, and zero-guest bookings could be real situations or data entry problems. I flagged or kept them instead of deciding which is true.
- **Assumptions about missing values.** Missing `children` values were treated as 0, and missing `country` values were labeled `"Unknown"`. Missing `agent` and `company` values are treated as "none recorded," but some may simply be missing data. This is especially true for the 3,167 TA/TO bookings with no agent.
- **Some issues are kept but not flagged.** The 180 zero-guest bookings and the single negative ADR booking stay in the data without a flag column, so I need to keep them in mind during analysis.
- **`Undefined` categories.** The meaning of `Undefined` in `meal`, `market_segment`, and `distribution_channel` isn't explained in the Kaggle description.
- **Limited scope.** The data covers only two hotels between July 2015 and August 2017, so results may not apply to other hotels or more recent years.

## 8. Repository Structure

```
├── dataset/
│   └── hotel_booking_cleaned.csv     # Cleaned dataset (119,390 rows × 41 columns)
├── notebooks/
│   └── 01_data_cleaning.ipynb        # Data quality assessment, cleaning, and preparation
└── README.md
```

The original `hotel_booking.csv` file is not included in this repository because it contains customer names, emails, phone numbers, and credit card numbers. It can be downloaded from the Kaggle link above.

## 9. Project Status and Next Steps

| Stage | Status |
|---|---|
| Data quality assessment | Completed |
| Data cleaning | Completed |
| Data preparation and export | Completed |
| Exploratory data analysis | Planned |
| Tableau dashboard | Planned |
| Business findings and recommendations | Planned |

Next, I plan to explore cancellation rates, booking lead times, seasonal demand, and ADR patterns in a new notebook. After that, I'll build a Tableau dashboard and write up my findings and recommendations.
