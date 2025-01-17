[<img src="https://github.com/QuantLet/Styleguide-and-FAQ/blob/master/pictures/banner.png" width="888" alt="Visit QuantNet">](http://quantlet.de/)

## [<img src="https://github.com/QuantLet/Styleguide-and-FAQ/blob/master/pictures/qloqo.png" alt="Visit QuantNet">](http://quantlet.de/) **BLEMmarketSimulation** [<img src="https://github.com/QuantLet/Styleguide-and-FAQ/blob/master/pictures/QN2.png" width="60" alt="Visit QuantNet 2.0">](http://quantlet.de/)

```yaml


Name of Quantlet: BLEMmarketSimulation

Published in: 'Forecasting in blockchain-based smart grids: Testing a prerequisite for the implementation of local energy markets'

Description: Simulates the market mechanism (blind double auction), which was implemented by Mengelkamp et al. (2018) in smart contract on a private Ethereum Blockchain, with real consumption and production data in three different supply scenarios (balanced, over-, and undersupply) and with true and predicted energy consumption values.

Keywords: blind double auction, market simulation, energy trading, zero-intelligence traders, smart grid, blockchain, smart contract, market mechanism, market outcome, equilibrium price, line graphs, time series, energy prediction, forecast, prediction error, energy cost, energy supply, energy demand

Author: Michael Kostmann

See also:
- BLEMdataGlimpse
- BLEMdescStatEnergyData
- BLEMevaluateEnergyPreds
- BLEMevaluateMarketSim
- BLEMplotEnergyData
- BLEMplotEnergyPreds
- BLEMplotPredErrors
- BLEMpredictLASSO
- BLEMpredictLSTM
- BLEMpredictNaive
- BLEMtuneLSTM
- BLEMplotScalingForLSTM

Submitted:  26.10.2018

Input:
- data: 100 consumer and 100 prosumer data sets containing electricity readings in 3-minute intervals (csv-files)
- predictions: 15-minute interval prediction of electricity consumption for every consumer forecasted with LASSO model (csv-file)

Output:
- market outcomes for each trading period (15-minute slots) in the three months testing period. I.e.,
- equilibrium price: the highest price which can still be served given the amounts offered by producers 
- local energy market price: average of equilibrium price and maximum price weighted by kWh amounts traded at the respective price
- bid results for every market participate acting as consumer:
     - consumer ID
     - amount specified by bidder
     - price bid
     - price paid (equilibrium price or maximum price depending on auction outcome)
     - cost in EURct paid by each consumer
- ask results for every market participate acting as producer:
     - producer ID
     - amount specified by sellers
     - price asked
     - price received (equilibrium price or minimum price depending on auction outcome)
     - revenue in EURct received by each producer
- total amount of electricity supplied in trading period
- total amount of electricity demanded in trading period
- oversupply (difference between total supply and total demand)

```

![Picture1](marketoutcome_pred_balanced.jpg)

![Picture2](marketoutcome_pred_oversupply.jpg)

![Picture3](marketoutcome_pred_undersupply.jpg)

![Picture4](marketoutcome_true_balanced.jpg)

![Picture5](marketoutcome_true_oversupply.jpg)

![Picture6](marketoutcome_true_undersupply.jpg)

![Picture7](producers_all.jpg)

### R Code
```r



## Plot energy production of all relevant prosumers in testing period
## Author: Michael Kostmann


# Load packages
packages = c("cowplot",
             "purrr")
invisible(lapply(packages, library, character.only = TRUE))

# Source user-defined functions
functions = c("FUN_getTargets.R",
              "FUN_generatePrices.R",
              "FUN_blindAuction.R")
invisible(lapply(functions, source))

# Function for easy string pasting
"%&%"     = function(x, y) {paste(x, y, sep = "")}

# Specify paths to directories containing consumer and prosumer datasets
path_c    = "../data/consumer/"
path_p    = "../data/prosumer/"

files_p    = substring(list.files(path_p, pattern = "*.csv"),
                       1, 17)[c(19, 24, 26, 30, 31, 72, 75, 83, 84, 85, 86, 89)]

# Load net production data per 15-minute time interval
prod = matrix(NA, nrow = 8836, ncol = length(files_p))
i = 0

for(id in files_p) {
    i = i+1
    prod[, i] = getTargets(path_p,
                           id,
                           return = "production",
                           min = "2017-10-01 00:03",
                           max = "2018-01-01 00:00")
}

# Vector of x-axis labels
ids = substring(files_p, 15, 17)

# Initiate list to save plots for plotting grid
plotgrids = list()

# Loop over specified producers
for(i in 1:length(files_p)){
    
    p_title =  ggdraw() + 
        draw_label("Prosumer "%&%ids[i],
                   size     = 10,
                   fontface = "bold")
    
    t = data.frame("time"   = seq.POSIXt(as.POSIXct("2017-10-01 00:00"),
                                         as.POSIXct("2018-01-01 00:00"),
                                         by = 900)[1:nrow(prod)],
                   "value" = prod[, i])
    
    p = ggplot(t, aes(x = time, y = value)) +
        geom_line() +
        scale_y_continuous(limits = c(0, max(prod))) +
        ylab("kWh/15 minutes") +
        xlab("timestamp") +
        theme_classic(base_size = 8)
    
    plotgrids[[i]] = plot_grid(p_title, p, ncol = 1, rel_heights = c(0.1, 1))
    
}

plot_grid(plotlist = plotgrids, nrow = 4, ncol = 3)
ggsave("producers_all.jpg", height = 8.267, width = 11.692)



###############################################################################



## Generate market outcomes with true and predicted values



# Clear global environment except for functions
rm(list = setdiff(ls(), lsf.str()))

###  Market simulation with true values  ###

# Specify paths to directories containing consumer and prosumer datasets
path_c  = "../data/consumer/"
path_p  = "../data/prosumer/"

# Specify which datasets from directories should be loaded for market analysis
files_c = substring(list.files(path_c,
                                pattern = "*.csv"),
                     1, 17)[-c(13, 21, 26, 35, 46, 53, 57, 67, 76, 78, 80, 81)]

# Balanced supply and demand
files_p = substring(list.files(path_p, pattern = "*.csv"),
                     1, 17)[c(19, 24, 26, 30, 72, 75, 89)] #, 31, 86, 83, 84, 85

# Oversupply
# files_p = substring(list.files(path_p, pattern = "*.csv"),
#                      1, 17)[c(19, 24, 26, 30, 72, 75, 83, 84, 89)] #, 31, 86, 85

# Undersupply
# files_p = substring(list.files(path_p, pattern = "*.csv"),
#                      1, 17)[c(19, 24, 26, 72, 75, 89)] # 30, 31, 86,  83, 84, 85


# Load consumption data per 15-minute time interval
cons = matrix(NA, nrow = 8836, ncol = (length(files_c)))
i = 0

for(id in files_c) {
    i = i+1
    cons[, i] = getTargets(path   = path_c,
                           id     = id,
                           return = "consumption",
                           min    = "2017-10-01 00:00",
                           max    = "2018-01-01 00:00")
}

# Load net production data per 15-minute time interval
prod = matrix(NA, nrow = 8836, ncol = length(files_p))
i = 0

for(id in files_p) {
    i = i+1
    prod[, i] = getTargets(path   = path_p,
                           id     = id,
                           return = "production",
                           min    = "2017-10-01 00:00",
                           max    = "2018-01-01 00:00")
}

# Generate prices
bid_prices = generatePrices(cons,
                             min_price = 12.31,
                             max_price = 28.69,
                             seed      = 20181005)

ask_prices = generatePrices(prod,
                             min_price = 12.31,
                             max_price = 28.69,
                             seed      = 20181005)

# Compuate market outcomes
market_outcomes_true = list()
pb = txtProgressBar(min = 0, max = nrow(cons), style = 3)

for(i in 1:nrow(cons)){
    market_outcomes_true[[i]] = blindAuction(cons_amount = cons[i, ],
                                             prod_amount = prod[i, ],
                                             max_price   = 28.69,
                                             min_price   = 12.31,
                                             bid_prices  = bid_prices[i, ],
                                             ask_prices  = ask_prices[i, ])
    setTxtProgressBar(pb, i)
}

# Save market outcomes
save(market_outcomes_true, file = "market_outcomes/true_outcomes_balanced.RData")
#save(market_outcomes_true, file = "market_outcomes/true_outcomes_oversupply.RData")
#save(market_outcomes_true, file = "market_outcomes/true_outcomes_undersupply.RData")



###  Market simulation with predicted values  ###

# Define vector of datasets to exclude from error analysis
remove   = c(13, 21, 34, 45, 52, 56, 66, 75, 77, 79, 81)

# Load predicted consumption values
cons_pred_all = read.csv("predictions/LASSO_predictions.csv")[, -1]
cons_pred     = cons_pred_all[, -remove]

# Compuate market outcomes
market_outcomes_pred = list()
pb = txtProgressBar(min = 0, max = nrow(cons_pred), style = 3)

for(i in 1:nrow(cons_pred)){
    market_outcomes_pred[[i]] = blindAuction(cons_amount = cons_pred[i, ],
                                             prod_amount = prod[i, ],
                                             max_price   = 28.69,
                                             min_price   = 12.31,
                                             bid_prices  = bid_prices[i, ],
                                             ask_prices  = ask_prices[i, ])
    setTxtProgressBar(pb, i)
}

# Save market outcomes
save(market_outcomes_pred, file = "market_outcomes/pred_outcomes_balanced.RData")
#save(market_outcomes_pred, file = "market_outcomes/pred_outcomes_oversupply.RData")
#save(market_outcomes_pred, file = "market_outcomes/pred_outcomes_undersupply.RData")



###############################################################################



## Plot oversupply and prices of market simulation


# Clear global environment except for functions
rm(list = setdiff(ls(), lsf.str()))

# Set vector
scenarios = c("_balanced", "_oversupply", "_undersupply")
captions  = c("Balanced supply", "Oversupply", "Undersuppy")

# Index for captions
c = 0

for(i in scenarios){
    
    # Load market outcomes
    load("market_outcomes/true_outcomes"%&%i%&%".RData")
    load("market_outcomes/pred_outcomes"%&%i%&%".RData")
    
    
    ## Market outcomes with true values ##
    
    # Extract oversupply and prices per period
    data_true = data.frame("time"       = 
                               seq.POSIXt(as.POSIXct("2017-10-01 00:00"),
                                          as.POSIXct("2018-01-01 00:00"),
                                          by = 900)[1:8836], 
                           "oversupply" = 
                               map_dbl(market_outcomes_true, "oversupply"),
                           "eq_price"   = 
                               map_dbl(market_outcomes_true, "eq_price"),
                           "LEM_price"  = 
                               map_dbl(market_outcomes_true, "LEM_price"))
    
    # Extract supply, demand, and equilibrium prices per period
    supply_true   = map_dbl(market_outcomes_true, "supply")
    demand_true   = map_dbl(market_outcomes_true, "demand")
    eq_price_true = map_dbl(market_outcomes_true, "eq_price")
    
    # Plot oversupply, equilibrium and weighted average prices
    c = c+1
    ptitle = ggdraw() +
        draw_label(captions[c]%&%": Market outcomes per trading period"%&%
                       "with true consumption values",
                   size     = 16,
                   fontface = "bold")
    
    p1 = data_true %>%
        ggplot() +
        geom_line(aes(x = time, y = oversupply)) +
        geom_hline(yintercept = 0, col = "red") +
        ylab("oversupply in kWh") +
        xlab("timestamp") +
        theme_classic(base_size = 10)
    
    p2 = data_true %>%
        ggplot() +
        geom_line(aes(x = time, y = eq_price)) +
        geom_hline(yintercept = 12.31, col = "red") +
        geom_hline(yintercept = 28.69, col = "red") +
        ylab("equilibrium price in EURct") +
        xlab("timestamp") +
        theme_classic(base_size = 10)
    
    p3 = data_true %>%
        ggplot() +
        geom_line(aes(x = time, y = LEM_price)) +
        geom_hline(yintercept = 12.31, col = "red") +
        geom_hline(yintercept = 28.69, col = "red") +
        ylab("LEM price in EURct") +
        xlab("timestamp") +
        theme_classic(base_size = 10)
    
    plot_grid(ptitle, p1, p2, p3, ncol = 1, rel_heights = c(0.15, 1, 1, 1))
    ggsave("marketoutcome_true"%&%i%&%".jpg",
           height = 8.267, width = 11.692)
    
    
    
    ## Market outcomes with predicted values ##
    
    # Extract oversupply and prices per period
    data_pred = data.frame("time"       = 
                               seq.POSIXt(as.POSIXct("2017-10-01 00:00"),
                                          as.POSIXct("2018-01-01 00:00"),
                                          by = 900)[1:8836],
                           "oversupply" = 
                               map_dbl(market_outcomes_pred, "oversupply"),
                           "eq_price"   =
                               map_dbl(market_outcomes_pred, "eq_price"),
                           "LEM_price"  =
                               map_dbl(market_outcomes_pred, "LEM_price"))
    
    # Extract supply, demand, and equilibrium prices per period
    supply_predb  = map_dbl(market_outcomes_pred, "supply")
    demand_pred   = map_dbl(market_outcomes_pred, "demand")
    eq_price_pred = map_dbl(market_outcomes_pred, "eq_price")
    
    # Plot oversupply, equilibrium and weighted average prices
    ptitle = ggdraw() +
        draw_label(captions[c],
                   size     = 16,
                   fontface = "bold")
    
    p1 = data_pred %>%
        ggplot() +
        geom_line(aes(x = time, y = oversupply)) +
        geom_hline(yintercept = 0, col = "red") +
        ylab("oversupply in kWh") +
        xlab("timestamp") +
        theme_classic(base_size = 10)
    
    p2 = data_pred %>%
        ggplot() +
        geom_line(aes(x = time, y = eq_price)) +
        geom_hline(yintercept = 12.31, col = "red") +
        geom_hline(yintercept = 28.69, col = "red") +
        ylab("equilibrium price in EURct") +
        xlab("timestamp") +
        theme_classic(base_size = 10)
    
    p3 = data_pred %>%
        ggplot() +
        geom_line(aes(x = time, y = LEM_price)) +
        geom_hline(yintercept = 12.31, col = "red") +
        geom_hline(yintercept = 28.69, col = "red") +
        ylab("LEM price in EURct") +
        xlab("timestamp") +
        theme_classic(base_size = 10)
    
    plot_grid(ptitle, p1, p2, p3, ncol = 1, rel_heights = c(0.15, 1, 1, 1))
    ggsave("marketoutcome_pred"%&%i%&%".jpg",
           height = 8.267, width = 11.692)
    
}


## end of file ##

```

automatically created on 2018-10-26