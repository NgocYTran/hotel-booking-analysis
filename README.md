# Hotel Booking Analysis

## 1. Project Overview

This project looks at hotel booking data for a city hotel and a resort hotel. I want to understand booking behavior, cancellations, and room pricing. The planned business questions are:

- How do bookings and cancellations vary by hotel, season, and booking channel?
- What factors are related to a booking being canceled?
- How does the average daily rate (ADR) change across hotels, room types, and months?
- Which market segments bring in the most estimated room revenue?

So far I have finished the data quality assessment, cleaning, and preparation stage. The exploratory analysis, Tableau dashboard, and business recommendations are still to come.

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
| `children` | 4 | All 4 bookings had 2–3 adults and no babies. | Filled with 0, assuming no children were recorded because there were none. |
| `country` | 488 | The missing values were spread across both hotels and several market segments, with no clear pattern. | Filled with `"Unknown"` so these bookings stay in the analysis without guessing a country. |
| `agent` | 16,340 | Most were Direct (7,625) or Corporate (5,543) bookings, which often don't involve a travel agent. However, 3,167 came through the TA/TO channel. | Left as missing. An agent ID can't be inferred, and filling with a placeholder could look like a real ID. |
| `company` | 112,593 | About 94% of bookings had no company ID. Most Online TA, Offline TA/TO, Groups, and Direct bookings had none, while most Aviation and Corporate bookings did. | Left as missing. A missing value most likely means the booking wasn't made through a company. |

I also checked how agents and companies relate to each other. Only 217 bookings had both an agent and a company ID. Most companies worked with only one agent. Company 281 was the exception and worked with 4.

### Duplicate Records

The raw dataset had **0 fully duplicated rows**. This is because every row has its own name, email, phone number, and credit card value. Once those columns were removed, many rows became identical (see [Final Dataset Validation](#6-final-dataset-validation)).

### Zero-Night and Zero-Guest Bookings

**Zero-night bookings:** 715 bookings had 0 weekend nights and 0 weeknights.

| Reservation status | Bookings |
|---|---:|
| Check-Out | 680 |
| Canceled | 22 |
| No-Show | 13 |

All 680 checked-out zero-night bookings had an ADR of 0, and their reservation status date was the same as the arrival date. They might be same-day stays or system entries, but the data doesn't say which. I kept them and left them out of the estimated revenue calculation.

**Zero-guest bookings:** 180 bookings had 0 adults, 0 children, and 0 babies. Most had an ADR of 0, but 31 were checked out with an ADR above 0. A booking can't really have zero guests, but I couldn't tell what the correct values should be, so I kept these rows unchanged.

### Negative and Zero ADR

- **Negative ADR:** Only 1 booking had a negative ADR (-6.38, Check-Out). A negative room rate doesn't make sense, so I kept the row but left it out of the estimated revenue calculation.
- **Zero ADR:** 1,959 bookings had an ADR of 0. The largest group was Complementary bookings (680), where a free stay is expected. Of the checked-out bookings with at least one night and an ADR of 0, **492 were not Complementary**. I couldn't explain these, so I flagged them instead of removing them.

### ADR Outliers (IQR Method)

ADR ranged from -6.38 to 5,400, with a mean of 101.83 and a median of 94.58. I used the interquartile range (IQR) method to find unusual values:

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
| `scheduled_arrival_date` | Combined arrival year, month, and day into one date | Makes time-based analysis and date comparisons easier |
| `total_guests` | `adults + children + babies` | Finds zero-guest bookings and measures group size |
| `total_nights` | `stays_in_weekend_nights + stays_in_week_nights` | Measures length of stay and finds zero-night bookings |
| `date_difference` | Days between `reservation_status_date` and `scheduled_arrival_date` | Checks whether status dates make sense for each reservation status |
| `estimated_room_revenue` | `adr × total_nights` | Gives a rough revenue estimate for completed stays |

`estimated_room_revenue` is only calculated for bookings that are **checked out**, have **at least one night**, and have an **ADR of 0 or higher**. All other rows are left blank. Canceled bookings and No-Shows didn't produce a completed stay, and zero-night or negative-ADR bookings would give misleading values. This leaves **74,485** bookings with an estimated revenue value. The median is 260.64 and the maximum is 7,590.

### Data Quality Flags

Instead of deleting questionable rows, I created three True/False flags. This keeps the full dataset, and in Tableau I can choose whether to include these rows.

| Flag | Rule | Rows flagged |
|---|---|---:|
| `adr_outlier_flag` | ADR is outside the IQR bounds | 3,793 |
| `unexplained_zero_adr_flag` | ADR = 0, checked out, at least one night, and not Complementary | 492 |
| `date_inconsistency_flag` | Status date is before the arrival date and the booking was not canceled | 1 |

## 6. Final Dataset Validation

- **Final shape:** 119,390 rows and 40 columns. No rows were removed. 4 customer columns were dropped and 8 new columns were added.
- **Remaining missing values (intentional):**

| Column | Missing | Reason |
|---|---:|---|
| `agent` | 16,340 | No agent recorded for the booking |
| `company` | 112,593 | No company recorded for the booking |
| `estimated_room_revenue` | 44,905 | Only calculated for checked-out bookings with at least one night and ADR ≥ 0 |

- **Apparent duplicates:** After the customer columns were removed, **31,994 rows** were identical to another row. I kept them because the dataset has no booking ID, so there's no reliable way to tell a true duplicate from two separate bookings with the same details, such as several identical rooms booked for one group. Removing them could undercount bookings and estimated revenue.
- **Data types:** `reservation_status_date` and `scheduled_arrival_date` were converted to dates, and the three flags are stored as True/False.
- **Export:** The cleaned data was saved as `dataset/hotel_booking_cleaned.csv`. I read the file back in and confirmed it has the same shape (119,390 × 40) and column names as the cleaned DataFrame.

## 7. Data Limitations

- **No unique booking ID.** I can't confirm whether the 31,994 apparent duplicates are repeated entries or separate bookings. If some are true duplicates, booking counts and revenue will be somewhat overstated.
- **Estimated room revenue is not actual revenue.** It is ADR multiplied by the number of nights. It doesn't include meals, extra charges, discounts, taxes, or cancellation fees, and it assumes the ADR applied to every night of the stay.
- **Unusual records aren't necessarily wrong.** High ADR values, zero-ADR stays, zero-night stays, and zero-guest bookings could be real situations or data entry problems. I flagged or kept them instead of deciding which is true.
- **Assumptions about missing values.** Missing `children` values were treated as 0, and missing `country` values were labeled `"Unknown"`. Missing `agent` and `company` values are treated as "none recorded," but some may simply be missing data. This is especially true for the 3,167 TA/TO bookings with no agent.
- **Some issues are kept but not flagged.** The 180 zero-guest bookings and the single negative ADR booking stay in the data without a flag column, so I need to keep them in mind during analysis.
- **`Undefined` categories.** The meaning of `Undefined` in `meal`, `market_segment`, and `distribution_channel` isn't explained in the Kaggle description.
- **Limited scope.** The data covers only two hotels between July 2015 and August 2017, so results may not apply to other hotels or more recent years.

## 8. Repository Structure

```
├── dataset/
│   └── hotel_booking_cleaned.csv     # Cleaned dataset (119,390 rows × 40 columns)
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
