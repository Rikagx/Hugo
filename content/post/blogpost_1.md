As a data practicioner in the public sector, I frequently work on data
compiled through surveys. We use surveys to get a feel for how clients
are reacting to various programming, to analyze qualitative and
quantitative data, and even to compile demographics. 90% of the time,
the data is collected through \[SurveyMonkey\] (www.surveymonkey.com) -
an online tool that works pretty well, especially if you upgrade to a
paid version. As with any online software tool that optimizes UI through
point and click tools, it has a few quirks.

If, like me, you want to do the analysis on your own or in some other
tool like R, I've found that there are a handful of best practices that
can help. I wanted to write a blogpost that went through some ways to
work through these quirks, summarize likert style data, and create some
easy, quick, and pretty visualizations.

### Exporting SM data

In the **Analyze Results** tab, SM allows users to download their data
in a variety of ways. You can download the following in csv, xlsx, pdf,
or spss outputs:

-   All summary data - I have not found this useful for my own analysis.
    This simply spits out the aggregate results of each question.

-   All responses data - exports the raw dataset with individual
    responses as rows and questions as columns. This is what I primarily
    use for my analysis.

-   All individual responses - exports all the answers for a specific
    individual.

### Cleaning data

In a standard survey, I will usually have a mix of demographic, binary,
and likert scale questions. When I load my data in R as a .csv, however,
I can quickly see that survey monkey has a funny way of organizing the
different questions and their associated labels. It looks like the
headers for the data are included over the first two rows of our data
table, depending on the type of question.

Binary questions have correct headers but then the first row for each
column has the character "Response".

    ## # A tibble: 4 x 4
    ##   `How do you identi~ `Are you a cigar~ `Which borough a~ `I am a high sch~
    ##   <chr>               <chr>             <chr>             <chr>            
    ## 1 Response            Response          Response          Response         
    ## 2 Prefer not to answ~ No                Manhattan         Yes              
    ## 3 Bisexual            No                Bronx             Yes              
    ## 4 Bisexual            No                Queens            Yes

Questions where the user can give multiple answer choices are spread
across several columns identified by unnamed headers X4, X5, X6, etc.

    ## # A tibble: 4 x 4
    ##   `What is your race?`  X4                       X5             X6         
    ##   <chr>                 <chr>                    <chr>          <chr>      
    ## 1 Black or African Ame~ Native Hawaiian or othe~ Caucasian      Hispanic o~
    ## 2 <NA>                  <NA>                     <NA>           <NA>       
    ## 3 <NA>                  <NA>                     <NA>           <NA>       
    ## 4 <NA>                  <NA>                     Non-Hispanic ~ <NA>

Likert questions have a blank header and our likert statement is
included in the 1st row of each column.

    ## # A tibble: 4 x 1
    ##   X14                                                                    
    ##   <chr>                                                                  
    ## 1 I feel comfortable telling staff any concerns I have about the program.
    ## 2 Neutral                                                                
    ## 3 Strongly agree                                                         
    ## 4 Agree

Finally, all of our factors are characters. Basically, its a mess!

Ok, so what now. My first step always, is to do a bit of basic cleaning
with the wonderful \[janitor\] (<https://github.com/sfirke/janitor>)
package. For the race category I want to combine the multiple choices
into one column, and I only want to know the first category identified
by each respondent. For this, the janitor package also has a handy
function called coalesce().

    library(janitor)
    mysurvey <- mysurvey %>% clean_names() %>% 
      mutate(race = coalesce(what_is_your_race, x4, x5, x6)) %>% 
      select(-what_is_your_race, -x4, -x5, -x6)  

    ## # A tibble: 10 x 1
    ##    race                     
    ##    <chr>                    
    ##  1 Black or African American
    ##  2 <NA>                     
    ##  3 <NA>                     
    ##  4 Non-Hispanic Caucasian   
    ##  5 Non-Hispanic Caucasian   
    ##  6 <NA>                     
    ##  7 <NA>                     
    ##  8 <NA>                     
    ##  9 Black or African American
    ## 10 <NA>

Next, whenever I have the meta data for a dataset separated up over
multiple rows, I like to generate a data dictionary so that I don't get
confused. I can also refer back to it, if and usually when something
goes wrong!

    names <- c("age", "gender", "so", "smoker", "borough", "hsd", "get_info", "file_complaint", "Q1", "Q2", "Q3", "Q4", "Q5", "race")
    data_dictionary <- mysurvey[1,] 
    names(data_dictionary) <- names
    data_dictionary <- data_dictionary %>% as.data.frame  %>% 
      rownames_to_column(., 'Var1') %>% select(-Var1) %>% 
      gather(key = "Question", value = "label", convert = TRUE)

    data_dictionary[7, 2] <- "I know where to get information on safety rights and/or reporting"
    data_dictionary[8, 2] <- "I know who to talk to or how to file a complaint if I feel unsafe"
    data_dictionary[14,2] <- "Response"

    data_dictionary

    ##          Question
    ## 1             age
    ## 2          gender
    ## 3              so
    ## 4          smoker
    ## 5         borough
    ## 6             hsd
    ## 7        get_info
    ## 8  file_complaint
    ## 9              Q1
    ## 10             Q2
    ## 11             Q3
    ## 12             Q4
    ## 13             Q5
    ## 14           race
    ##                                                                      label
    ## 1                                                                 Response
    ## 2                                                                 Response
    ## 3                                                                 Response
    ## 4                                                                 Response
    ## 5                                                                 Response
    ## 6                                                                 Response
    ## 7        I know where to get information on safety rights and/or reporting
    ## 8        I know who to talk to or how to file a complaint if I feel unsafe
    ## 9       Overall, I am very satisfied with my participation in the program.
    ## 10 I feel comfortable telling staff any concerns I have about the program.
    ## 11                             Staff do a good job of running the program.
    ## 12                   There are enough staff members to keep everybody safe
    ## 13                    Staff members are properly trained to meet my needs.
    ## 14                                                                Response

So I did a slightly weird thing here. I know I'm going to be doing
analysis with my likert statements, but I'd really prefer not having to
type out the entire statement or having it be included in my summary
tables. Instead, I renamed all the likert statements to a simple
alphanumeric, and kept the actual statement in the label column. Also,
if your other variables include interesting response choices or things
that you'd like to note for yourself, you can specify them in the label
column here. I really like having something like this, even if it
includes a touch more work.

Next we need to actually clean up the first two rows of our dataframe.
Since we already have our list of column names from creating the data
dictionary, all we need to do is rename the dataframe, remove the 2nd
row of meta-data, and parse the columns accordingly. And now we have a
dataset that we can work with!

    names(mysurvey) <- names
    mysurvey <- mysurvey[-1,]   

    mysurvey$age <- parse_number(mysurvey$age)
    cols <- colnames(mysurvey[ ,2:14])
    mysurvey[,cols] <- lapply(mysurvey[,cols], as.factor)

    glimpse(mysurvey)

    ## Observations: 485
    ## Variables: 14
    ## $ age            <dbl> 20, 18, 19, 19, 20, 20, 19, 19, 19, 17, 20, 20,...
    ## $ gender         <fct> Male, Female, Female, Male, Male, Male, Male, M...
    ## $ so             <fct> Prefer not to answer, Bisexual, Bisexual, Prefe...
    ## $ smoker         <fct> No, No, No, No, No, No, No, No, No, No, No, Yes...
    ## $ borough        <fct> Manhattan, Bronx, Queens, Brooklyn, Staten Isla...
    ## $ hsd            <fct> Yes, Yes, Yes, Yes, Yes, Yes, Yes, Yes, Yes, Ye...
    ## $ get_info       <fct> Yes, Yes, Yes, Yes, Yes, Yes, Yes, Yes, Yes, Ye...
    ## $ file_complaint <fct> Yes, Yes, Yes, Yes, Yes, Yes, Yes, Yes, Yes, Ye...
    ## $ Q1             <fct> Neutral, Strongly agree, Agree, Agree, Strongly...
    ## $ Q2             <fct> Neutral, Strongly agree, Agree, Agree, Strongly...
    ## $ Q3             <fct> Neutral, Strongly agree, Agree, Agree, Strongly...
    ## $ Q4             <fct> Neutral, Strongly agree, Agree, Disagree, Agree...
    ## $ Q5             <fct> Neutral, Strongly agree, Agree, Agree, Agree, A...
    ## $ race           <fct> NA, NA, Non-Hispanic Caucasian, Non-Hispanic Ca...
