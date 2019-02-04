R Markdown
----------

As a data practicioner in the public sector, I frequently work on data
compiled through surveys. We use surveys to get a feel for how clients
are reacting to various programming, to analyze qualitative and
quantitative data, and even to compile demographics. 90% of the time,
the data is collected through \[SurveyMonkey\] (www.surveymonkey.com) -
an online tool that works pretty well, especially if you upgrade to a
paid version. As with any online software tool that optimizes UI through
point and click tools, it has a few quirks. If, like me, you want to do
the analysis on your own or in some other tool like R,I've found that
there are a handful of best practices that can help. I wanted to write a
blogpost that went through some ways to work through these quirks,
summarize likert style data, and create some easy, quick, and pretty
visualizations.

### Exporting SM data

In the **Analyze Results** tab, SM allows users to download their data
in a variety of ways. You can download the following in csv, xlsx, pdf,
or spss outputs:

\*All summary data - I have not found this useful for my own analysis.
This simply spits out the aggregate results of each question.

\*All responses data - exports the raw dataset with individual responses
as rows and questions as columns. This is what I primarily use for my
analysis.

\*All individual responses - exports all the answers for a specific
individual.

### Cleaning data

In a standard survey, I will usually have a mix of demographic, binary
(Yes or No), and likert scale questions. When I load my data in R,
however, I can quickly see that SM has a funny way of organizing the
questions and their associated labels, and answer choices.

    mysurvey <- read_csv(here("mysurvey.csv"))

    ## Parsed with column specification:
    ## cols(
    ##   `What is your age?` = col_character(),
    ##   `What is your gender?` = col_character(),
    ##   `What is your race?` = col_character(),
    ##   X4 = col_character(),
    ##   X5 = col_character(),
    ##   X6 = col_character(),
    ##   `How do you identify your sexual orientation?` = col_character(),
    ##   `Are you a cigarette smoker?` = col_character(),
    ##   `Which borough are you from?` = col_character(),
    ##   `I am a high school graduate` = col_character(),
    ##   `I know where to get information on safety, rights and/or reporting?` = col_character(),
    ##   `I know who to talk to or how to file a complaint if I am worried about my safety or the behavior of a particular staff member.` = col_character(),
    ##   `Please indicate how strongly you agree or disagree with the following statements:` = col_character(),
    ##   X14 = col_character(),
    ##   X15 = col_character(),
    ##   X16 = col_character(),
    ##   X17 = col_character()
    ## )

    glimpse(mysurvey)

    ## Observations: 486
    ## Variables: 17
    ## $ `What is your age?`                                                                                                              <chr> ...
    ## $ `What is your gender?`                                                                                                           <chr> ...
    ## $ `What is your race?`                                                                                                             <chr> ...
    ## $ X4                                                                                                                               <chr> ...
    ## $ X5                                                                                                                               <chr> ...
    ## $ X6                                                                                                                               <chr> ...
    ## $ `How do you identify your sexual orientation?`                                                                                   <chr> ...
    ## $ `Are you a cigarette smoker?`                                                                                                    <chr> ...
    ## $ `Which borough are you from?`                                                                                                    <chr> ...
    ## $ `I am a high school graduate`                                                                                                    <chr> ...
    ## $ `I know where to get information on safety, rights and/or reporting?`                                                            <chr> ...
    ## $ `I know who to talk to or how to file a complaint if I am worried about my safety or the behavior of a particular staff member.` <chr> ...
    ## $ `Please indicate how strongly you agree or disagree with the following statements:`                                              <chr> ...
    ## $ X14                                                                                                                              <chr> ...
    ## $ X15                                                                                                                              <chr> ...
    ## $ X16                                                                                                                              <chr> ...
    ## $ X17                                                                                                                              <chr> ...

The main issues in the export are that:

-race seems to be a multi-answer category, so SM includes the answers
across multiple columns -The first two rows include meta-data unevenly.
FOr demographic or binary questions, the question "label" is in the
first row. But for likert scale questions, the statement that
respondents agree or disagree with, is in the second row, since they are
all subsets of the main question "Please indicate how strongly you agree
or disagree with the following statements." In these situations, the
first row is blank (e.g. X14, X15, X16, X17) and the second row includes
the statement that we want to see. -All my factors are categories

First, I'll fix the multi-answers for race and do a bit of basic
cleaning and renaming with the janitor package.

    library(janitor)
    mysurvey <- mysurvey %>% clean_names() %>% 
      mutate(race = coalesce(what_is_your_race, x4, x5, x6)) %>% 
      select(-what_is_your_race, -x4, -x5, -x6) %>% 
      glimpse()

    ## Observations: 486
    ## Variables: 14
    ## $ what_is_your_age                                                                                                              <chr> ...
    ## $ what_is_your_gender                                                                                                           <chr> ...
    ## $ how_do_you_identify_your_sexual_orientation                                                                                   <chr> ...
    ## $ are_you_a_cigarette_smoker                                                                                                    <chr> ...
    ## $ which_borough_are_you_from                                                                                                    <chr> ...
    ## $ i_am_a_high_school_graduate                                                                                                   <chr> ...
    ## $ i_know_where_to_get_information_on_safety_rights_and_or_reporting                                                             <chr> ...
    ## $ i_know_who_to_talk_to_or_how_to_file_a_complaint_if_i_am_worried_about_my_safety_or_the_behavior_of_a_particular_staff_member <chr> ...
    ## $ please_indicate_how_strongly_you_agree_or_disagree_with_the_following_statements                                              <chr> ...
    ## $ x14                                                                                                                           <chr> ...
    ## $ x15                                                                                                                           <chr> ...
    ## $ x16                                                                                                                           <chr> ...
    ## $ x17                                                                                                                           <chr> ...
    ## $ race                                                                                                                          <chr> ...

Now we have the top answers for race only in one column! Next, I like to

    names <- c("age", "gender", "so", "smoker", "borough", "hsd", "get_info", "file_complaint", "Q1", "Q2", "Q3", "Q4", "Q5", "race")

    data_dictionary <- mysurvey[1,] 
    names(data_dictionary) <- names

    data_dictionary <- data_dictionary %>% as.data.frame  %>% 
      rownames_to_column(., 'Var1') %>% select(-Var1) %>% 
      gather(key = "Question", value = "label", convert = TRUE)
      
    data_dictionary <- data_dictionary[1,7] <- 
    data_dictionary <- data_dictionary[1,8]
    data_dictionary <- data_dictionary[1,14]

Since likert questions tend to be long sentences, I usually just rename
them as Q\# and keep the labels separate in a data dictionary. This
makes analysis and visualizing charts much easier. The questions in the
data disctionary were also the ones were the labels were missing from
the 1st row - so this makes organizing the data much easier.
