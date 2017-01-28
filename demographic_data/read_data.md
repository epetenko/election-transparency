Add Demographic Variables to Registration and Voting Data
=========================================================

The code below is an example of adding demographic data to our voting and registration data. At this point I have only added age by sex and race variables from the 2015 5-year American Community Survey data. However, the code below can be relatively easily adapted to pull anything else from the ACS.

What you will need to get started:
----------------------------------

-   A Census API Token, if you don't already have one it is super easy. Go here to request: <http://api.census.gov/data/key_signup.html>
-   To use the below code you will need the updated uselections R package, which can be downloaded here using the devtools package and the command `devtools::install_github("Data4Democracy/election-transparency/r-packages/uselections")`

``` r
library(dplyr, quietly = TRUE)
library(uselections, quietly = TRUE)  # I am pulling the data from the R package, could use data.world
library(knitr, quietly = TRUE)
library(acs, quietly = TRUE)
library(reshape2, quietly = TRUE)
library(stringr, quietly = TRUE)
library(tidyr, quietly = TRUE)
```

Pulling and cleaning the data
-----------------------------

First we need to pull the data from the Census API. The below code pulls data at the County level by age, race, sex, and Hispanic origin.

``` r
api.key.install(key = '378bfe7e9514b505b44c7aff5702dc081046b38b') # You will need to enter an API key

geokey <- geo.make(state = "*", 
                   county = "*")

# Set list of attributes you want to pull, you will want to look for the table numbers. 


pull_list <- c("B01001A",    # Sex by Age: White Alone
               "B01001B",    # Sex by Age: Black Alone
               "B01001C",    # Sex by Age: 
               "B01001D",    # Sex by Age:
               "B01001E",
               "B01001F",
               "B01001G",
               "B01001H",    # Sex by Age: White Alone Non-Hispanic
               "B01001I")

# Define function to pull data and dump into a data frame
pull <- function(pull){
  df <- acs.fetch(endyear = 2015, span = 5, geography = geokey, table.number = pull, 
            col.name = "pretty")
  new_df <- data.frame(paste0(str_pad(df@geography$state, 2, "left", pad="0"), 
                               str_pad(df@geography$county, 3, "left", pad="0")), 
                        df@estimate, 
                        stringsAsFactors = FALSE)
  names(new_df)[1] <- 'geoid'
  return(new_df)
}

#Apply this function over our list of tables, the result will be a list of DFs
dfs <- lapply(pull_list, pull)

# split out the non-hispanic white alone because it does not have the same number of categories
# from the API
df_nhwa <- dfs[[8]]
dfs <- dfs[-8]

demo_frame <- data.frame(NULL)

# Join together all of your tables. I am doing it this way because all of my tables have the same 
# dimensions and are measured the same way. If I had tables with different demographic dimensions
# I would want to clean them separately, so group and join your tables accordingly.
for (i in 1:length(dfs)){
  if (i == 1){
    demo_frame <- dfs[[i]]
  } else{
    demo_frame <- demo_frame %>% inner_join(dfs[[i]], by = "geoid")
  }
}
```

Now that we have all of our data in on big data frame with a ton of columns, I want to work on getting something more usable. For these age, sex, race, Hispanic origin data I will give us the component parts to be able to calculate other important variables. Accordingly, I leave the data separated by sex / race / Hispanic origin, but group age into 3 categories:

1.  Under 18 Years Old
2.  18 to 64 Years Old
3.  65 Years and Older

We may want to make different decisions in the future, but I think this is a good start.

``` r
# Start with the data set with everything but the Non-Hispanic White Alone data
demo_frame1 <- demo_frame %>%
  melt(id = "geoid") %>%
  separate(variable, into = c("variable", "race", "sex", "age"), fill = "right", sep = "\\.\\.") %>%
  select(-2) %>%
  filter(!is.na(age)) %>%
  dcast(geoid + race + sex ~ age, value.var = "value") %>%
  mutate(under18 = Under.5.years + `5.to.9.years` + `10.to.14.years` + `15.to.17.years`,
         votingAge = `18.and.19.years` + `20.to.24.years` + `25.to.29.years` +
           `30.to.34.years` + `35.to.44.years` + `45.to.54.years` + `55.to.64.years` 
         + `65.to.74.years` + `75.to.84.years` + `85.years.and.over`) %>%
  select(-(4:17)) %>%
  mutate(sex = ifelse(sex == ".Female", "female", "male"),
         race = ifelse(race == "AMERICAN.INDIAN.AND.ALASKA.NATIVE.ALONE", "aian",
                ifelse(race == "ASIAN.ALONE", "asian", 
                ifelse(race == "BLACK.OR.AFRICAN.AMERICAN.ALONE", "black",
                ifelse(race == "HISPANIC.OR.LATINO", "hisp",
                ifelse(race == "NATIVE.HAWAIIAN.AND.OTHER.PACIFIC.ISLANDER.ALONE", "nhpi",
                ifelse(race == "SOME.OTHER.RACE.ALONE", "sor",
                ifelse(race == "TWO.OR.MORE.RACES", "multi", "white")))))))) %>%
  melt() %>%
  dcast(geoid + race + variable ~ sex) %>%
  mutate(pop = male + female) %>%
  select(-(4:5)) %>%
  dcast(geoid ~ race + variable)

# Clean the Non-Hispanic White Alone data
demo_frame2 <- df_nhwa %>%
  melt(id = "geoid") %>%
  separate(variable, into = c("variable", "race", "race2", "sex", "age"), 
           fill = "right", sep = "\\.\\.") %>%
  select(-2, -4) %>%
  filter(!is.na(age)) %>%
  dcast(geoid + race + sex ~ age, value.var = "value") %>%
  mutate(under18 = Under.5.years + `5.to.9.years` + `10.to.14.years` + `15.to.17.years`,
         votingAge = `18.and.19.years` + `20.to.24.years` + `25.to.29.years` +
           `30.to.34.years` + `35.to.44.years` + `45.to.54.years` + `55.to.64.years` 
         + `65.to.74.years` + `75.to.84.years` + `85.years.and.over`) %>%
  select(-(4:17)) %>%
  mutate(sex = ifelse(sex == ".Female", "female", "male"),
         race = "nhwa") %>%
  melt() %>%
  dcast(geoid + race + variable ~ sex) %>%
  mutate(pop = male + female) %>%
  select(-(4:5)) %>%
  dcast(geoid ~ race + variable) %>%
  inner_join(demo_frame1, by = "geoid") %>%
  filter(as.numeric(geoid) < 72000)
```

The race categories are labelled as: 1. nhwa: Non-Hispanic White Alone 2. aian: American Indian or Alaska Native 3. asian: Asian 4. black: Black or African American 5. hisp: Hispanic or Latino (Any Race) 6. multi: 2 or More Races 7. nhpi: Native Hawaiian or other Pacific Islander 8. sor: Some other Race 9. white: White

To get the total population sum the following categories: \* aian \* asian \* black \* multi \* nhpi \* sor \* white