# Statistical Analysis of Effects of Weather Conditions on Consumer Behavior and Return Rates

**Final Report**
**Kaan Bayri**

---

## 1. Motivation

The idea that weather affects human mood and behavior is something most people have felt at some point, but in the context of online shopping it is rarely studied with real data. I wanted to test whether this everyday intuition holds up when you actually measure it: does bad weather make people order more often, spend more per order, or end up canceling their orders at a higher rate?

More specifically, the question I started with was whether bad weather (rain, unexpected cold) triggers impulsive purchases driven by boredom or low mood, and whether those impulsive orders show a higher tendency to be canceled afterwards. If the answer is yes, that has real implications for e-commerce companies: weather forecasts could feed into promotional timing, and the returns and cancellation pipeline could be planned around expected weather patterns instead of treated as random.

Beyond the topic itself, this project was also a chance to practice a full end-to-end data pipeline. Going from nine separate raw tables and a 60 million row weather file all the way to clustering, hypothesis testing, and a supervised model is the kind of work I had only done in isolated pieces before. Doing it on a question I was genuinely curious about made it much easier to push through the messier parts.

## 2. Data Source

The project integrates two completely separate public datasets that I had to figure out how to connect.

**Olist Brazilian E-Commerce Dataset.** A relational database released by Olist, a major Brazilian e-commerce marketplace. It covers orders placed between 2016 and 2018 and is split across nine tables: orders, customers, order items, payments, products, product category translations, geolocation, sellers, and reviews. Each order has a timestamp, a customer ZIP code prefix, an order status (delivered, canceled, etc.), and the items and payment details associated with it. I downloaded it from Kaggle, where Olist hosts it publicly.

**INMET Automatic Weather Stations Dataset.** A time series dataset from Brazil's National Institute of Meteorology (INMET) containing hourly readings of temperature, precipitation, humidity, pressure, wind, and other variables from automatic stations across the country for 2000 to 2021. I used the 2016 to 2018 subset to match the Olist period. This is also a public dataset, available through INMET's open data portal.

Neither dataset was collected by me directly; both are public records. The real challenge was that the two datasets share no common key. Olist gives customer locations as Brazilian ZIP code prefixes; INMET gives weather station coordinates. Linking them required a spatial join, which I describe in the next section.

## 3. Data Analysis

The analysis was carried out across seven Jupyter notebooks, each building on the previous one. This section walks through what each stage did and the techniques used.

### 3.1 Data Cleaning

The first notebook reduced the raw data to only what the analysis needed. From the Olist orders table I kept order id, customer id, status, and purchase timestamp. The other columns (delivery dates, approval timestamps, carrier dates) were dropped. From the order items table I aggregated to total spend per order using a groupby. From products I kept only the English category name, after merging with the translation table. The sellers and reviews tables were dropped entirely since the focus was on buyer behavior, not seller performance or sentiment.

The INMET file was the hardest part of cleaning. The full file is around 60 million hourly rows and roughly 14 GB on disk, far too large to load with a single read_csv call on a normal laptop. I read it in chunks of 500,000 rows and filtered each chunk down to the 2016 to 2018 period before keeping it, which cut the data down to a manageable size. I also kept only temperature, precipitation, and humidity columns; pressure, radiation, dew point, wind speed and direction were dropped.

A few data quality issues needed handling. The INMET file uses -9999.0 as a missing value sentinel, which would silently destroy any statistical analysis if treated as a real reading, so I replaced it with NaN. Hourly weather was aggregated to daily averages (mean temperature, total rainfall, mean humidity) using groupby on station and date. One weather station coded as CRIOSFERA had coordinates in Antarctica, clearly wrong for a Brazil-only dataset, and was removed.

### 3.2 Spatial and Temporal Joining

This was the most demanding engineering step. The pipeline goes order to customer to ZIP code to coordinates to nearest weather station to weather reading on that day.

The first two merges were straightforward joins on customer id and on ZIP code prefix. The spatial join was not. For every unique ZIP code in the customer base I needed to find the closest weather station out of all of them. Doing this per order would mean 88,000 distance calculations across hundreds of stations each. Doing it per unique ZIP first and then merging the result back into the orders table reduced the work dramatically while giving the same answer.

For the distance calculation itself I used a simple Euclidean formula on latitude and longitude, basically the Pythagorean theorem on the coordinate differences. My original proposal mentioned the Haversine formula, which accounts for the curvature of the earth and is technically more accurate. In practice, for the relatively short distances between a Brazilian customer and their nearest weather station, both formulas pick the same nearest station almost every time, and the Euclidean version runs significantly faster. I treated this as an acceptable tradeoff for the scale of the project.

Once each order had a station id attached, I merged with the daily weather table on (station_id, purchase_date) to attach temperature, rainfall, and humidity for the exact day and location of every order. Orders with no weather match (no station data on that day) or no spend data were dropped. The final master table had 88,675 orders and 23 columns, and is the input to every notebook from here on.

### 3.3 Exploratory Data Analysis

Before doing any tests I looked at the data to check that it made sense. The full date range turned out to be September 2016 to September 2018, with order volume growing steeply across the period. The mean order value was R$139 and the median was R$87, which is a clear right skew driven by a small number of very expensive orders. Temperature on order days ranged from -2.4 to 35.6 degrees Celsius. About 60 percent of order days had zero rainfall, and the rest had some.

The most striking single number was the cancellation rate: 0.46 percent, or 407 cancellations out of 88,675 orders. This low base rate would turn out to matter a lot for the supervised model later.

I also bucketed temperature into three groups (cold, mild, hot) and computed the cancellation rate in each. Cold days had a cancellation rate of 0.55 percent compared to 0.39 percent on hot days, a noticeable difference. I followed up on this in the hypothesis testing notebook. The full correlation matrix between daily order count, average spend, and weather variables showed only weak correlations (all absolute values below 0.1), which was an early signal that any effects would be small in magnitude even where they were statistically real.

### 3.4 Hypothesis Tests

Three questions were tested formally:

* Does weather affect the number of orders placed per day?
* Does weather affect how much each order is worth?
* Does weather affect the cancellation rate?

For the first two I used two-sample t-tests, which compare the means of two groups and return a p-value indicating how likely the observed difference is to be random. For the third I used a chi-square test on a contingency table of cancelled versus not cancelled across weather groups, which is the right test for comparing proportions rather than averages. The significance threshold was the conventional p < 0.05.

Bad weather was defined two ways in this notebook: rainy (precipitation > 0 mm) versus dry, and cold (below median temperature) versus hot. The results, summarized in one place:

| Test | Weather split | p-value and verdict |
|---|---|---|
| Order volume per day | Rainy vs dry | p = 0.006, significant |
| Order volume per day | Cold vs hot | p = 0.797, not significant |
| Spend per order | Rainy vs dry | p = 0.551, not significant |
| Spend per order | Cold vs hot | p < 0.001, significant |
| Cancellation rate | Rainy vs dry | p = 0.749, not significant |
| Cancellation rate | Cold vs hot | p < 0.001, significant |

Three of the six tests came back significant. Rain matters for whether people order at all; temperature matters for how much they spend and how often they cancel. The cold-day cancellation finding is the one that motivated the deeper analysis in notebook 07, where I later realized this result was partially driven by a seasonal confound.

### 3.5 K-Means Customer Segmentation

The goal here was to find customer groups that behave differently in relation to weather. I built four features per customer:

* **Recency:** days since their last order
* **Frequency:** total number of orders
* **Monetary:** average order value
* **Weather sensitivity:** percentage of their orders placed on rainy days (precipitation > 5 mm)

The first finding came before the algorithm even ran. Every customer in the Olist dataset has exactly one order, so the frequency column has a standard deviation of zero. Olist is a marketplace dominated by one-off purchases, and the classic RFM "F" dimension is dead in this dataset. I kept the feature in for transparency about the modeling choices, but it contributes nothing to the clustering. Recency, monetary value, and weather sensitivity had to carry the entire signal.

Features were standardized with StandardScaler, and K-means was run for k = 2 through 9. I evaluated each k with the elbow method on inertia and with silhouette score on a 5,000-row sample (running silhouette on all 88,000 rows is computationally expensive). The silhouette was highest at k=2 (0.541), but k=4 (0.509) gave more interpretable segments for an e-commerce context, so I went with four clusters.

The four resulting segments:

| Segment | Customers | Avg recency (days) | Avg spend (R$) | Weather % |
|---|---|---|---|---|
| Recent casual buyer | 42,013 | 122 | 135 | 0% |
| Inactive casual buyer | 30,444 | 390 | 135 | 0% |
| High spender | 2,152 | 237 | 1,192 | 13% |
| Weather-triggered buyer | 14,065 | 259 | 139 | 100% |

The weather-triggered segment is the most interesting one for this project. These are 14,065 customers whose entire purchase history happened on rainy days. Whether they actually behave differently from the casual buyers is the question the remaining notebooks try to answer.

### 3.6 Random Forest Cancellation Model

To see whether weather mattered for predicting cancellations, I trained a Random Forest classifier on order status (delivered versus canceled). The features included weather (temperature, precipitation, humidity), order details (price, item count, freight cost, purchase hour, day of week, weekend flag), customer segment from the clustering step, and the top 10 product categories one-hot encoded (with everything else lumped as "other"). Total of 24 features.

Two things had to be confronted upfront. First, the dataset is severely class-imbalanced: only 0.46 percent of orders are cancelled. A model that just predicts "delivered" for every order is 99.5 percent accurate but completely useless. I used class_weight='balanced' to make the model treat each cancellation as worth roughly 200 times a delivered order during training. Even so, with only 401 cancellation examples to learn from across 87,000 rows, the model could not reliably distinguish individual cancellations. Recall on the canceled class was effectively zero on the held-out test set.

So the raw predictions were not the goal. The real output of this notebook is the feature importance ranking, which describes which inputs the trees relied on most when splitting cancelled from delivered orders. Weather features came out very high:

| Feature | Rank (of 24) | Importance |
|---|---|---|
| humidity_mean | 3 | 0.144 |
| temp_mean_c | 4 | 0.141 |
| precip_total_mm | 6 | 0.064 |

All three weather features land in the top 6 out of 24. Order value and freight cost rank slightly higher, which makes sense (expensive orders are more likely to be reconsidered), but weather is clearly in the mix.

Independently of the model, I computed the cancellation rate directly per segment, which does not depend on any modeling assumptions:

| Segment | Cancellation rate |
|---|---|
| High spender | 1.00% |
| Weather-triggered buyer | 0.51% |
| Inactive casual buyer | 0.45% |
| Recent casual buyer | 0.42% |

Weather-triggered buyers cancel about 21 percent more often than regular casual buyers (0.51 percent versus 0.42 percent). The absolute numbers are tiny, but the relative gap is consistent with the impulse-and-regret hypothesis.

### 3.7 Impulse Buying Analysis

This was the most analytically important notebook and the one where the central project question actually got tested. The original idea was that bad weather drives impulsive purchasing, which should leave fingerprints in the data: a shift toward indoor and comfort categories, more late-night ordering, higher freight tolerance, more credit card use, more big-ticket items, and more regret-driven cancellations.

While running this analysis I caught a problem with the earlier hypothesis tests. Notebook 04 defined "cold" using absolute temperature (below overall median). But absolute cold is mostly just winter. Buying a blanket on a cold day in July (Brazilian winter) is seasonal shopping, not impulse. So the cold-day cancellation effect from notebook 04 might be picking up regular seasonal patterns rather than weather-driven impulse.

To fix this, I redefined bad weather as either more than 5 mm of rainfall, or a temperature anomaly more than 3 degrees below that month's average. This isolates *unexpected* cold, which is the thing that could plausibly catch people off guard and shift their mood. This is also the right framing for any production marketing system: an unexpectedly cold Tuesday in October is a real event worth reacting to, while a cold Tuesday in July is just July.

With the seasonality-adjusted definition, I tested five impulse signals:

| Signal | Good weather | Bad weather | Relative change |
|---|---|---|---|
| Indoor/comfort category share | 15.8% | 15.9% | essentially flat |
| Freight to price ratio (median) | 0.226 | 0.223 | -1.6% |
| Credit card share | 75.6% | 75.1% | -0.7% |
| High-value orders (>R$300) | 8.3% | 7.8% | -5.7% |
| Cancellation rate | 0.425% | 0.515% | +21.2% |

Only one signal survives the seasonality correction: cancellation rate is 21.2 percent higher on bad weather days. This is the regret signature of impulse buying. People order, then reconsider once the weather clears. The other signals (category shift, freight tolerance, credit card use, high-value share, purchase hour) show no real change once seasonality is removed.

I also broke out average spend by segment and by weather. High spenders go up about 9 percent on anomalous bad weather days (R$1,164 to R$1,269), while casual buyers' spend drops slightly. So the small impulse effect on spend, to the extent it exists, is concentrated in the high-spender segment rather than spread evenly across customers.

## 4. Findings

Summarizing across all seven notebooks, the headline results are:

**Order volume responds to rain but not to temperature.** Rainy and dry days have significantly different order volumes (p = 0.006). Temperature shows no effect on volume.

**Spend per order responds to temperature but not to rain.** Orders placed on hot days are worth significantly more than orders placed on cold days (R$143.70 vs R$133.43, p < 0.001). Rain has no measurable effect on average spend.

**Cancellations respond to bad weather, but the effect is real and modest.** Using absolute temperature, cold-day orders are cancelled more often (0.54% vs 0.38%, p < 0.001). After controlling for seasonality, bad-weather orders still cancel 21.2 percent more often than good-weather orders (0.515% vs 0.425%). This is the strongest piece of evidence for impulse buying followed by regret.

**Customer segmentation reveals a "weather-triggered" group.** Of 88,675 customers, 14,065 made 100 percent of their purchases on rainy days. These customers cancel at 0.51 percent, noticeably higher than the 0.42 percent rate of regular casual buyers.

**Weather features are top predictors of cancellation but cannot drive a useful predictive model.** Humidity, temperature, and precipitation rank 3, 4, and 6 out of 24 features in the Random Forest. The model itself cannot reliably classify individual cancellations because of class imbalance, but feature importance confirms that weather is one of the strongest discriminators the model finds.

**The comfort buying or nesting hypothesis does not hold once seasonality is controlled.** I initially expected indoor and comfort categories to gain share on bad weather days. They don't (15.8 percent to 15.9 percent, essentially flat). The apparent shift when using absolute cold turned out to be a seasonal artifact.

**High spenders go up, casual buyers go slightly down, in anomalous bad weather.** Average order value for high spenders rises about 9 percent on anomalous bad weather days. This is the only segment-level spend signal worth acting on for marketing.

Taken together, the project supports a narrower version of the original hypothesis than I started with. Weather genuinely affects e-commerce behavior, but the effect is bounded: rain shifts how many orders happen, temperature shifts how much each one is worth, and bad weather modestly increases regret-driven cancellations. The dramatic "boredom buying" story I was originally testing largely does not survive a seasonality-adjusted analysis, except in the cancellation signature where it does hold up.

## 5. Limitations and Future Work

The single biggest constraint on this project was the one-time-buyer problem. In the Olist dataset, roughly 96 percent of customers have exactly one order in their entire history. This wrecks the F (frequency) dimension of the standard RFM segmentation model: there is literally no variance to cluster on, since every customer has frequency = 1. I kept the feature in the clustering for transparency, but it contributed nothing, and the segmentation had to lean entirely on recency, monetary value, and weather sensitivity.

The consequence is that I could only compare across customers (the weather-triggered segment versus casual buyers), not within them. The really interesting question, whether the same customer behaves differently on bad weather days than on good weather days, is unanswerable with this dataset. A repeat-customer e-commerce dataset would let me ask much sharper questions: are weather-sensitive customers consistently weather-sensitive over time? Do their bad-weather orders cancel at a higher rate than their good-weather orders, controlling for everything else?

Unfortunately, large-scale public e-commerce datasets with multi-purchase customer histories are extremely rare. Most retailers consider this kind of data competitively sensitive and don't release it. Olist is one of the very few publicly available end-to-end e-commerce datasets, which is why it shows up in so many academic projects despite its limitations. A larger or richer dataset would substantially improve this project, and finding or negotiating access to one is probably the single highest-impact change I could make to this analysis.

Other limitations worth flagging:

**Cancellation class imbalance.** Only 401 cancellations across 87,000 orders. The Random Forest could not learn to predict individual cancellations reliably at this rate, no matter how good the features were. Any model that tries to predict cancellation as a target is fundamentally bounded by how few positive examples exist. A larger dataset with more cancellations would let the model itself become useful rather than just providing feature importance rankings.

**Distance approximation.** I used Euclidean distance on raw latitude and longitude rather than the Haversine formula mentioned in my proposal. For nearest-station matching inside Brazil the practical answers are nearly identical, but the simpler formula is technically less correct over large distances. A cleaner future version should use Haversine, or even better, a KD-tree for faster nearest-neighbor queries.

**One weather reading per order.** Each order was matched to a single daily weather summary at the customer's nearest station. This ignores within-day variation (someone ordering at 9 pm on a day that started sunny and ended in a storm) and ignores weather before the purchase, which is probably what actually influences mood. A future iteration could use hourly weather matched to the exact purchase hour, or the average weather of the preceding 24 to 72 hours.

**Geographic scope.** The analysis is Brazil-only. Brazilian seasons and consumer behavior may not generalize. Replicating the same pipeline with European or US data would test whether the effects I found are universal or culturally and climatically specific.

**No causal claims.** Every test in this project is correlational. Even with significant p-values, I cannot say weather causes changes in behavior, only that the patterns are associated. A proper causal analysis would need either natural experiments (comparing nearby regions where one had unexpected rain and one didn't on the same day) or a difference-in-differences design.

Future extensions worth pursuing:

* Match each order to the weather of the prior 24 to 48 hours instead of the purchase day itself, which is probably closer to the actual mood-driving variable.
* Apply the same pipeline to a multi-purchase dataset and analyze within-customer behavior shifts across weather conditions.
* Use a more interpretable model (logistic regression with regularization, or SHAP values on the Random Forest) to quantify exactly how much each weather variable changes cancellation odds.
* Pull in review sentiment from the Olist reviews table (which I dropped early in cleaning) and check whether bad-weather orders that did not get cancelled still received worse reviews. That would be another way to test the regret hypothesis without needing an actual cancellation event.
* Build a small dashboard or visualization frontend that lets a marketing team explore the segment behavior interactively. The dataset and pipeline are now clean enough to support this.

## Closing Note

The full project, including all seven notebooks, the cleaned data outputs, and the figures referenced in this report, is available in the project Git repository. Every notebook runs top to bottom with the cleaned data and reproduces the numbers and figures cited here.
