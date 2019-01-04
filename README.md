-   [Objective](#objective)
-   [Prework](#prework)
-   [Structure of the data to use](#structure-of-the-data-to-use)
-   [Exploratory analysis](#exploratory-analysis)
-   [Predicting tomorrow's traffic volume on each toll booth](#predicting-tomorrows-traffic-volume-on-each-toll-booth)

### Objective

This project consists on analyzing the evolution of traffic on AUSA toll booths in Buenos Aires highways. The data being used in this project can be found in the [Buenos Aires Data](https://data.buenosaires.gob.ar/dataset/flujo-vehicular-por-unidades-de-peaje-ausa) site. In particular, the project aims to predict the amount of vehicles going in and out of the city for a given date in the future. So, given a day *d*<sub>*i*</sub> the model will be able to predict the traffic volume on day *d*<sub>*i* + 1</sub>.

### Prework

-   Unzip the `Flujo-Vehicular-por-Unidades-de-Peaje-AUSA.zip` file provided by the [Buenos Aires Data](https://data.buenosaires.gob.ar/dataset/flujo-vehicular-por-unidades-de-peaje-ausa) site.
-   Run the `standardize.values()` function from `utils.R`.

### Structure of the data to use

The information provided consists of 11 files, one for each year from 2008 to 2018 (which only provides information until September 9th). There is no strict convention on the columns naming, neither on whether categorical values are stored with uppercase letters or a mixture of uppercase and lowercase. To standardize everything to the same criteria, several transformations were performed. Those transformations are defined in `src/utils.R` and consist of:

-   `standardize.columns.and.merge_`: not all of the files have the same column names, nor the same column order. This function standardizes their format, and merges them all in a single .csv file.
-   `standardize.values`: among categorical attributes, not all of them have the same value for the same concept, for example you could find the same toll name writen in all caps and then all lowers. This function standardizes that, and creates a new dataset.

The .csv files provide in each entry an estimate of the amount of vehicles that went through a certain toll both in an interval of time of one hour. The following is the detail of each column:

-   `Y`: indicates the **year** number.
-   `M`: indicates the **month** number.
-   `D`: indicates the **day** number.
-   `day.name`: indicates the **day of the week**.
-   `start.hour`: indicates the **start time of the interval**.
-   `end.hour`: indicates the **end time of the interval**.
-   `toll.booth`: indicates the **name of the toll booth**.
-   `vehicle.type`: indicates whether the vehicle was a **truck or a car**.
-   `payment.method`: indicates if the **payment method**.
-   `amount`: indicates the **number of vehicles** that went through the toll booth at that interval of time, and payed with a given payment method.

In addition to the previously mentioned columns, two new columns were added to include the geographic position of each toll booth via coordinates, `LAT` and `LONG`. The values of each coordinate were not provided by the dataset, and had to be manually searched using Google Maps.

For example:

``` r
rm(list = ls()); gc();
# Include the definitions of the methods used throughout the notebook.
source('./src/flow.R')
df <- read.csv('~/Documents/traffic/datasets/traffic.csv')
head(df)
```

    ##   day.name start.hour end.hour toll.booth vehicle.type payment.method
    ## 1   MARTES   00:00:00 01:00:00    ALBERDI      LIVIANO       EFECTIVO
    ## 2   MARTES   00:00:00 01:00:00 AVELLANEDA      LIVIANO       EFECTIVO
    ## 3   MARTES   00:00:00 01:00:00 DELLEPIANE      LIVIANO       EFECTIVO
    ## 4   MARTES   00:00:00 01:00:00      ILLIA      LIVIANO       EFECTIVO
    ## 5   MARTES   01:00:00 02:00:00    ALBERDI      LIVIANO       EFECTIVO
    ## 6   MARTES   01:00:00 02:00:00 AVELLANEDA      LIVIANO       EFECTIVO
    ##   amount       LAT      LONG    Y M D Q
    ## 1      7 -34.64480 -58.49205 2008 1 1 1
    ## 2     71 -34.64827 -58.47811 2008 1 1 1
    ## 3     34 -34.65047 -58.46561 2008 1 1 1
    ## 4     27   0.00000   0.00000 2008 1 1 1
    ## 5     37 -34.64480 -58.49205 2008 1 1 1
    ## 6    345 -34.64827 -58.47811 2008 1 1 1

### Exploratory analysis

#### Increment in traffic

As a way of starting to know this dataset, we can see there's a difference in the amount of entries for each file in our dataset.

``` r
volume.per.year <- df %>%
    group_by(Y) %>% 
    summarise(total.volume = sum(amount))

p <- ggplot(volume.per.year, 
            aes(volume.per.year$Y, volume.per.year$total.volume)) +
  geom_point(shape = 1) +
  stat_summary(fun.data = mean_cl_normal) +
  labs(x = 'Time', y = 'Number of vehicles') +
  geom_smooth(method = 'lm') +
  scale_x_continuous(labels = function(x) ceil(x)) +
  theme_bw()

plot(p)
```

![](README_files/figure-markdown_github/trend-1.png)

From the graph above it's clear there's an order of magnitud of difference between the amount of vehicles in the period 2008-2013 versus the period 2014-2018. This is the result of two factors:

-   An increment in the amount of observations for tool booths like those of Avellaneda and Alberdi.

``` r
t <- df[c('toll.booth', 'Y', 'amount')] %>%
  group_by(toll.booth, Y) %>%
  summarise(total.volume = sum(amount))

p <- ggplot(data = t,
            aes(x = Y, y = total.volume, group = toll.booth)) +
  geom_line(aes(linetype = toll.booth, color = toll.booth)) +
  labs(x = NULL, y = NULL, title = 'Amount of vehicles per year') +
  scale_x_continuous(labels = function(x) ceil(x)) +
  theme_bw()

plot(p)
```

![](README_files/figure-markdown_github/incrementinobservations-1.png)

-   The addition of new toll booths around 2014 like those of Sarmiento, Retiro and Salguero as illustrated below.

``` r
theme_set(theme_classic())

t <- df[c('toll.booth', 'Y', 'amount')] %>%
  group_by(toll.booth, Y) %>%
  summarise(amount = sum(amount)) %>%
  mutate(
    start_year = min(Y),
    end_year = max(Y)) %>%
  distinct(toll.booth, start_year, end_year)

p <- ggplot(data = t, aes(x = start_year, xend = end_year, 
                          y = toll.booth, group = toll.booth)) +
  geom_dumbbell(color = '#0e668b', colour_x = '#0e668b', size_x = 1.5, 
                colour_xend = '#a3c4dc', size_xend = 0.75) +
  labs(x = NULL, y = NULL, 
       title = 'Addition of new toll booths to the dataset') +
  scale_x_continuous(labels = function(x) ceil(x)) +
  theme(plot.title = element_text(hjust = 0.5, face = 'bold'),
        plot.background = element_rect(fill='#ffffff'),
        panel.background = element_rect(fill='#ffffff'),
        panel.grid.minor = element_blank(),
        panel.grid.major.y = element_blank(),
        panel.grid.major.x = element_line(),
        axis.ticks = element_blank(),
        legend.position = 'top',
        panel.border = element_blank())
plot(p)
```

![](README_files/figure-markdown_github/timeframes-1.png)

#### Hourly patterns in toll booths

Traffic flow holds a pattern for each toll booth. Performing an hourly breakdown of traffic for the Alberdi tool booth, shown in the following graph, two traffic peaks can be identified: one around 8am and another one around 6pm. This matches with the times people is commuting to an from work. Another interesting pattern is that traffic keeps relatively still between the time of the two peaks.

``` r
t <- custom.agg(df[(df$toll.booth == 'ALBERDI'),],
                length(unique(df$Y)),
                'vehicle.type', 'start.hour')
p <- ggplot(data = t, aes(x = format(strptime(t$start.hour,"%H:%M:%S"),'%H'), 
                          y = amount, group = vehicle.type)) +
    labs(title = 'Vehicles per hour in Alberdi toll booth', 
         x = 'start.hour',
         y = 'Number of vehicles') +
    geom_line(aes(linetype = vehicle.type, color = vehicle.type)) +
    theme_bw()
plot(p)
```

![](README_files/figure-markdown_github/breakdownalberdi-1.png)

The previous graph differenciates between the two different kinds of vehicles: motorbikes/cars and trucks. While both kinds show a similar behavior, the volume of motorbikes/cars is greater than the one of trucks. Although this behavior can be seen in other toll booths like Avellaneda or Sarmiento, there are cases like Retiro, where the amount of trucks is almost the same as the one of cars.

``` r
t <- custom.agg(df[(df$toll.booth == 'RETIRO'),],
                length(unique(df$Y)),
                'vehicle.type', 'start.hour')
p <- ggplot(data = t, aes(x = format(strptime(t$start.hour,"%H:%M:%S"),'%H'), 
                          y = amount, group = vehicle.type)) +
    labs(title = 'Vehicles per hour in Retiro toll booth', 
         x = 'start.hour',
         y = 'Number of vehicles') +
    geom_line(aes(linetype = vehicle.type, color = vehicle.type)) +
    theme_bw()
plot(p)
```

![](README_files/figure-markdown_github/breakdownretiro-1.png)

Even though the volume of vehicles going through the Retiro toll booth is approximatelly three times smaller than the one of Alberdi, the ratio Liviano-Pesado (light-heavy) is greater. The reason for this is that Retiro is a a commercial route, located in the road to the Port of Buenos Aires, reason why the volume of heavy vehicles is greater than in other toll booths.

Another interesting aspect to analyze of this dataset is the distribution of drivers commiting infractions (id est, not paying when they cross the toll booth), and whether this is something that only happens for light vehicles or in heavy vehicles too. Below are two charts with the distribution of infractions for both Alberdi and Retiro.

``` r
t <- custom.agg(df[(df$toll.booth == 'ALBERDI' & 
                      df$payment.method == 'INFRACCION'),],
                length(unique(df$Y)),
                'vehicle.type', 'start.hour')
p <- ggplot(data = t, aes(x = format(strptime(t$start.hour,"%H:%M:%S"),'%H'), 
                          y = amount, group = vehicle.type)) +
    labs(title = 'Infractions per hour in Alberdi toll booth', 
         x = 'start.hour',
         y = 'Number of vehicles') +
    geom_line(aes(linetype = vehicle.type, color = vehicle.type)) +
    theme_bw()
plot(p)
```

![](README_files/figure-markdown_github/infractionsalberdi-1.png)

``` r
t <- custom.agg(df[(df$toll.booth == 'RETIRO' & 
                      df$payment.method == 'INFRACCION'),],
                length(unique(df$Y)),
                'vehicle.type', 'start.hour')
p <- ggplot(data = t, aes(x = format(strptime(t$start.hour,"%H:%M:%S"),'%H'), 
                          y = amount, group = vehicle.type)) +
    labs(title = 'Infractions per hour in Retiro toll booth', 
         x = 'start.hour',
         y = 'Number of vehicles') +
    geom_line(aes(linetype = vehicle.type, color = vehicle.type)) +
    theme_bw()
plot(p)
```

![](README_files/figure-markdown_github/infractionsretiro-1.png)

In both cases, the distribution of infractions seems to follow the same distribution of traffic throughout the day: the moments where there are more infractions correlate with those in which traffic volume is the highest.

Another interesting phenomenom are users moving from paying in cash (EFECTIVO) to using automatic tolls (AUPASS). Looking at the animation below, 2016 starts showing a decline in the number of vehicle owners paying in *cash*. However, since 2014 the use of *automatic tolls* started to steadily increase. As a note, the decline in 2018 on both cash and automatic tolls is due to the fact that 2018 data is still not complete.

``` r
t <- df %>%
  group_by(payment.method, Y) %>%
  filter(payment.method %in% c(
    'EFECTIVO', 'AUPASS', 'NO COBRADO', 'INFRACCION')) %>%
  summarise(total.volume = sum(amount))

a <- ggplot(data = t,
            aes(Y, total.volume, group = payment.method)) +
  geom_line(aes(linetype = payment.method, color = payment.method)) +
  geom_segment(aes(xend = 2018, yend = total.volume), 
               linetype = 2, colour = 'grey') +
  geom_point(size = 2) +
  scale_y_log10() +
  geom_text(aes(x = 2018.2, label = payment.method), hjust = 0) +
  transition_reveal(Y) +
  coord_cartesian(clip = 'off') + 
  labs(title = 'Evolution of top 4 payment methods along the years',
       x = NULL, y = 'Total amount of payments') +
  scale_x_continuous(labels = function(x) ceil(x)) +
  theme_minimal() +
  theme(plot.margin = margin(5.5, 60, 5.5, 5.5), 
        legend.position='none')

animate(a)
```

![](README_files/figure-markdown_github/paymentsperyearanimated-1.gif)

### Predicting tomorrow's traffic volume on each toll booth

TODO(tulians): Use the `forecast` library to predict the next 2 days traffic volume.
