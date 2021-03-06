---
title: "4. showtime"
author: "Ali Ezzat"
date: "February 4, 2019"
output: html_document
---



## Load necessary data


```r
get_xml_db_rows('L:/full database.xml')
```

```
## [1] TRUE
```

```r
drug_dta <- parse_drug()
dti_dta <- parse_drug_targets()
target_dta <- parse_drug_targets_polypeptides()
#ddi_dta <- parse_drug_interactions()
drug_exp_prop_dta <- parse_drug_experimental_properties()
drug_class_dta <- parse_drug_classifications()
drug_group_dta <- parse_drug_groups()
drug_targets_actions <- parse_drug_targets_actions()

drug_dta <- drug_dta %>% unique
dti_dta <- dti_dta %>% unique
target_dta <- target_dta %>% unique
#ddi_dta <- ddi_dta %>% unique
drug_exp_prop_dta <- drug_exp_prop_dta %>% unique
drug_class_dta <- drug_class_dta %>% unique
drug_group_dta <- drug_group_dta %>% unique
drug_targets_actions <- drug_targets_actions %>% unique
```


## Impressive Plots


```r
## get approved drugs list
approved_drugs <- 
    drug_group_dta %>% 
    filter(text == 'approved')
approved_drugs <- approved_drugs$parent_key
withdrawn_drugs <-     
    drug_group_dta %>% 
    filter(parent_key %in% approved_drugs) %>% 
    filter(text == 'withdrawn')
withdrawn_drugs <- withdrawn_drugs$parent_key
approved_drugs <- setdiff(approved_drugs, withdrawn_drugs)
```


```r
## molecular weight: approved vs. unapproved drugs
mol_weight_dta <- 
    drug_exp_prop_dta %>% 
    filter(kind == 'Molecular Weight') %>% 
    select(-kind, -source) %>% 
    mutate(value = as.double(value)) %>% 
    mutate(status = ifelse(parent_key %in% approved_drugs,'approved','not approved')) %>% 
    as_data_frame

mol_weight_dta %>% 
    ggplot(aes(status, value, fill = status)) +
    geom_boxplot() + 
    scale_y_log10()
```

![plot of chunk unnamed-chunk-3](figure/unnamed-chunk-3-1.png)





```r
## status in drug development pipeline

## get all actual groups
get_all_actual_groups <- function(x) {
    ## apply the 'get_actual_group' function to each drug
    x <- sapply(x, get_actual_group)
    
    ## return the actual groups for all drugs
    x
}

## get actual group for a single drug
get_actual_group <- function(x) {
    ## withdrawn > approved > investigational > experimental > other
    if (grepl('withdrawn', x)) {               actual_group <- 'withdrawn'
    } else if (grepl('approved', x)) {         actual_group <- 'approved'
    } else if (grepl('investigational', x)) {  actual_group <- 'investigational'
    } else if (grepl('experimental', x)) {     actual_group <- 'experimental'
    } else {                                   actual_group <- 'other'}
    
    ## return actual final tag for the drug
    actual_group
}


final_drug_group_dta <- 
    drug_group_dta %>% 
    group_by(parent_key) %>%
    summarise(allgroups = paste(text, collapse=';')) %>%
    mutate(final_group = get_all_actual_groups(allgroups)) %>% 
    select(-allgroups) %>% 
    rename(status = final_group)

drug_dta %>% 
    rename(parent_key = primary_key) %>% 
    left_join(final_drug_group_dta, by = 'parent_key') %>%
    group_by(created) %>% 
    summarise(approved = sum(status == 'approved'),
              experimental = sum(status == 'experimental'),
              investigational = sum(status == 'investigational'),
              withdrawn = sum(status == 'withdrawn'),
              other = sum(status == 'other')) %>% 
    rename(creationDate = created) %>% 
    ggplot(aes(x = creationDate)) + 
    geom_line(aes(y = cumsum(approved), col = 'approved')) + 
    geom_line(aes(y = cumsum(experimental), col = 'experimental')) + 
    geom_line(aes(y = cumsum(investigational), col = 'investigational')) + 
    geom_line(aes(y = cumsum(withdrawn), col = 'withdrawn')) + 
    geom_line(aes(y = cumsum(other), col = 'other')) + 
    #scale_y_continuous(limits = c(0,12000)) + 
    xlab('Time') + 
    ylab('Quantity') + 
    labs(title = 'Drug Status',
         subtitle = 'Proportions of the different status values over time', 
         caption = 'created by ggplot')
```

![plot of chunk unnamed-chunk-4](figure/unnamed-chunk-4-1.png)



