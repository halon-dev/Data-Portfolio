# Cyclistic Insights: Boosting Memberships Through Data Analytics

## Project Background
Cyclistic is a public bike-share program operating in Chicago since 2016, offering a network of over 5,800 bicycles and more than 600 docking stations citywide. The fleet includes a variety of bike types—standard, adaptive, and cargo models—designed to serve a broad population, including riders with physical disabilities.

The service accommodates two primary user types: casual riders, who pay per ride or per day, and annual members, who commit to longer-term usage. Casual riders tend to use the service for leisure, while members are more likely to rely on it for commuting and regular transportation.

Historically, Cyclistic has used a wide-reaching marketing strategy to attract users. However, internal financial reviews have shown that long-term members are more valuable to the company in terms of revenue and engagement. As a result, the marketing director, Lily Moreno, has tasked the analytics team with identifying behavioral differences between casual riders and members. The objective is to support a targeted campaign that encourages more casual users to transition into annual memberships through insights drawn from trip usage patterns and rider trends.

- **Category 1:** Usage Behavior  
- **Category 2:** Ride Duration and Time-of-Day Patterns  
- **Category 3:** Station and Route Trends  
- **Category 4:** User Segmentation and Business Value

---

## Data Structure & Initial Checks

The main database structure included two tables (`q1` and `q4`), which were merged into a single `full_data` table after column and type alignment. Each table represented one quarter of trip data and was assessed for uniqueness, nulls, and consistency. All transformations were documented and backed up.

---

## Executive Summary

**Overview of Findings:**
- Members account for 86% of trips and show a strong preference for commuting hours.
- Casual riders hold trips 4x longer and peak during leisure hours and weekends.
- Differences in station usage, trip duration, and behavior highlight strong segmentation potential.

---

## Insights Deep Dive

### Category 1: Usage Behavior
- **Trip Share:** Members made 86% of all trips, casual riders 14%.
- **Time of Day:** Members peaked during traditional commute hours (7–9 AM, 3–5 PM). Casuals peaked mid-afternoon.
- **Day of Week:** Members were more active Monday–Friday. Casual riders peaked Saturday–Sunday.
- **Ride Duration Bands:** Most trips were under 30 minutes; members skewed <15 minutes.

### Category 2: Ride Duration and Time-of-Day Patterns
- **Average Duration:** Casual riders averaged nearly 4x the ride length of members.
- **Outliers:** Duration outliers beyond 1744 minutes (upper 3 standard deviations) were removed. 82% were casual riders.
- **Invalid Data:** Negative ride lengths and maintenance trips were removed.

### Category 3: Station and Route Trends
- **Station Usage:** No overlap between top 3 stations for casuals vs. members.
- **Route Popularity:** Popular routes suggest different travel goals (commute vs. leisure).
- **Data Issues:** Station 208 had multiple names; reassigned based on majority label.

### Category 4: User Segmentation and Business Value
- **User Labels:** Reclassified `Subscriber` to `Member` and `Customer` to `Casual`.
- **Pattern Recognition:** Members show high consistency and predictability.
- **Marketing Angle:** Casual riders are strong targets for seasonal promotions, especially at peak locations.
- **Operational Insight:** Inconsistent station IDs flagged for internal review.

---

## Recommendations

- **Promote memberships through weekend and afternoon offers**, targeting casual riders where they peak.
- **Optimize marketing around top casual stations** with discounts or upgrade messaging.
- **Push upgrade notifications at 30–60 minute mark** during rides to promote longer-term commitment.
- **Use route and station data to forecast growth areas** for docking expansions and service improvements.

---

## Assumptions and Caveats

- Assumed station ID 208 with inconsistent names referred to the same physical location.
- Converted all timestamps using `STR_TO_DATE` with standard format assumptions.
- Deleted negative ride lengths and maintenance routes as invalid.
- Removed outliers manually due to SQL delete constraints on subqueries.

---

## Resources
- [SQL Queries Used for Cleaning](SQL_Cleaning.md)
- [SQL Queries for Insights](SQL_Analysis.md)
- [Interactive Tableau Dashboard](Tableau_Link)

