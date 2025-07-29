---
title: "Enabling Personalization at Scale"
#excerpt: "Improvements to the intelligence underlying a practice test delivery system (“Problem Roulette”).<br/><img src='/images/500x300.png'>"
excerpt: "Improving the intelligence underlying a practice test delivery system.<br/><img src='/files/port-pr-streaks-counter.png' style='width:75%;'>"
collection: portfolio
---

One of my machine learning projects was to work with software, behavioral science, and product teams to improve the intelligence underling an educational technology tool called <a href="https://problemroulette.ai.umich.edu/Welcome/" target="_blank">Problem Roulette</a> - an online quiz delivery system built and hosted by the Center for Academic Innovation. The end goals of the project were to (1) serve students personalized study tips, and (2) to enable adaptive question delivery.

## (1) PERSONALIZED STUDY TIPS

The first step to serving personlized study tips was to decide what information to provide study tips for. Decision Tree modeling was used for this decision. Dozens of behaviors in Problem Roulette (page views, latency, number of questions answered, accuracy, etc.) were compared to determine which were most strongly related to course grade. The top three are listed below. 

    Study Volume (total number of questions completed)
    Accuracy (total number correct)
    Question Difficulty (fraction of correct responses to total attempts)

With a list of key behaviors in hand, the next step was to understand their inter-relationships so that personalized messaging can be developed. Figure 1 was helpful in this regard. It shows the conditions under which a particular behavior was more or less effective at impacting course grade. For example, it can be seen from the steep slopes of the green lines that adding study volume is very beneficial when accuracy is high, but is considerably less beneficial when accuracy is lower, which can be seen in the flatter slopes of the red and blue lines. Accordingly, we would not want to nudge students with low accuracy to study more (volume will not helpful) but rather to study more intentionally (accuracy will be helpful).

<!--  FIGURE: PROFICIENCY SCORE DIMENSIONS  -->
Figure 1. Mean course grade (GPE) by study volue (number of questions completed), user accuracy, and question difficulty.
<img src="/files/port-pr-dimensions.svg" alt="PR Dimensions" style="max-width: 100%; height: auto;">

**Key Takeaways** 

(1) **More questions help**, so long as you’re getting them correct (i.e., more positive slope for high accuracy (green) than medium (red) or low (blue)). **More questions hurt** GPE when answering incorrectly.

(2) **More accurate answers almost always help**. Accuracy matters less the fewer number of questions completed.

(3) **Question difficulty is not super important on its own**, which can be seen by the similar heights of each colored line. If anything, this plot suggests that harder questions are a bit more helpful (each colored line is a bit higher than the corresponding one as questions get easier).

These key takeaways were then used by the Behavioral Science Team to generate personalized study tips. For example, if a user was answering a high number of easy questions with high accuracy, they would receive a tip to study harder questions. If a user was answering a high number of hard questions with low accuracy, they would receive a tip to study more intentionally. If a user were answering a medium number of medium difficulty questions with medium accuracy, they would receive a tip to study more questions of higher difficulty with more intentionality. Ultimately, users received tips on the dimension(s) that they could improve upon.

<!--  LINK TO THE PR BLOG  -->
<a href="https://onlineteaching.umich.edu/articles/teaching-students-to-fish-problem-roulette-empowers-online-students-to-become-self-sufficient-learners/" target="_blank" rel="noopener noreferrer">
  Read more in my article for the University of Michigan's Online Teaching website 
</a> <br>

## (2) ADAPTIVE QUESTION DELIVERY

**Adaptive testing** is an assessment approach where question difficulty dynamically adjusts in real time based on a test-taker’s responses. In Problem Roulette, this approach allows students to focus on the right number and difficulty of questions for their learning goals, rather than working through a fixed set.

The first step in enabling adaptive testing is to accurately measure the **difficulty of each question**. These difficulty scores provide the question-selection algorithm with the data it needs to choose the next question, ensuring students are consistently challenged — neither overwhelmed by overly hard problems nor bored by material they’ve already mastered. The system also tracks individual performance trends and incorporates a feedback loop to refine both student proficiency estimates and question difficulty ratings over time, creating a personalized, data-driven learning environment.

The SQL pipeline below supports this system by consolidating raw question and session data, aggregating performance at the student, question, and course levels, and applying quality filters to ensure reliability. Its final outputs — question-level difficulty scores and user-level proficiency estimates — feed directly into the adaptive testing engine, enabling dynamic, personalized question selection.


## Pipeline Overview

Below is the **full SQL script** that develops a reproducible SQL pipeline for computing **question difficulty scores** in Problem Roulette.

The script:
1. Builds a unified dataset (`userQS`) from user question and session data.  
2. Aggregates performance at the **student**, **question**, and **course** levels.  
3. Applies **data quality filters** (minimum attempts, users, and accuracy).  
4. Computes **difficulty scores for each question** and **average difficulty for each user**.

---

## Step 1. Pull raw data, join, and filter out missing responses.
```sql
WITH userQS AS (
  SELECT 
      s.course_id,
      q.id AS user_question_id,
      q.question_id,
      q.user_id,
      q.correct
  FROM problemroulette.problem_roulette_userquestion q
  LEFT JOIN problemroulette.problem_roulette_usersession s 
         ON q.user_session_id = s.id
  WHERE q.correct IS NOT NULL
),
```

## Step 2. Create summary tables that hold aggregated student-, question-, and course-level data needed for defining filter thresholds.

```sql
-- Student-level summary tracks each student's attempts and performance by question.
level_student AS (
  SELECT 
      course_id,
      user_id,
      question_id,
      COUNT(correct) AS NumAttempts,
      SUM(correct) AS NumCorrect,
      SUM(correct)::decimal / COUNT(correct) AS PctCorr,
      CASE WHEN SUM(correct) > 0 THEN 1 ELSE 0 END AS any_correct
  FROM userQS
  GROUP BY course_id, user_id, question_id
),

-- Question-level summary aggregates attempts and accuracy across all students.
level_question AS (
  SELECT 
      course_id,
      question_id,
      SUM(NumAttempts) AS question_NumAttempts,
      SUM(NumCorrect) AS question_NumCorrect,
      SUM(NumCorrect)::decimal / SUM(NumAttempts) AS question_PctCorr
  FROM level_student
  GROUP BY course_id, question_id
),

-- Course-level summary counts questions and unique users per course.
level_course AS (
  SELECT 
      course_id,
      COUNT(DISTINCT question_id) AS course_NumQuestions,
      COUNT(DISTINCT user_id) AS course_NumUsers
  FROM userQS
  GROUP BY course_id
),
```

## Step 3. Merge student-, question-, and course-levels summaries and set up filters.
```sql
-- Merge summary data.
PR_level_sum AS (
  SELECT 
      s.*,
      q.question_NumAttempts,
      q.question_NumCorrect,
      q.question_PctCorr,
      c.course_NumQuestions,
      c.course_NumUsers,
      q.question_NumAttempts::decimal / c.course_NumUsers AS question_frac_attempts
  FROM level_student s
  LEFT JOIN level_question q 
         ON s.course_id = q.course_id AND s.question_id = q.question_id
  LEFT JOIN level_course c 
         ON s.course_id = c.course_id
),

-- Filter 1: Minimum attempts per question (≥10). 
filter1 AS (
  SELECT 
      course_id, 
      question_id,
      CASE WHEN question_NumAttempts < 10 THEN 1 ELSE 0 END AS filter1
  FROM PR_level_sum
),

-- Filter 2: Minimum unique users (≥10). 
filter2_counts AS (
  SELECT 
      course_id,
      question_id,
      COUNT(DISTINCT user_id) AS question_NumDistinctUsers
  FROM PR_level_sum
  GROUP BY course_id, question_id
),
filter2 AS (
  SELECT 
      course_id, 
      question_id,
      CASE WHEN question_NumDistinctUsers < 10 THEN 1 ELSE 0 END AS filter2
  FROM filter2_counts
),

-- Filter 3: Minimum accuracy threshold (≥25% correct).
filter3_counts AS (
  SELECT 
      course_id,
      question_id,
      COUNT(user_id) AS question_NumUsersAns,
      SUM(any_correct) AS question_NumUsersAns_Corr,
      SUM(any_correct)::decimal / COUNT(user_id) AS question_PctUsersCorr
  FROM PR_level_sum
  GROUP BY course_id, question_id
),
filter3 AS (
  SELECT 
      course_id, 
      question_id,
      CASE 
        WHEN question_PctUsersCorr = 0 THEN 1
        WHEN question_PctUsersCorr > 0 AND question_PctUsersCorr < 0.25 THEN 1
        ELSE 0
      END AS filter3
  FROM filter3_counts
),

-- Merge filters into userQS.
PR_filters AS (
  SELECT DISTINCT
      pr.course_id,
      pr.question_id,
      f1.filter1,
      f2.filter2,
      f3.filter3
  FROM PR_level_sum pr
  LEFT JOIN filter1 f1 ON pr.course_id=f1.course_id AND pr.question_id=f1.question_id
  LEFT JOIN filter2 f2 ON pr.course_id=f2.course_id AND pr.question_id=f2.question_id
  LEFT JOIN filter3 f3 ON pr.course_id=f3.course_id AND pr.question_id=f3.question_id
),
userQS_with_filters AS (
  SELECT 
      u.*,
      (COALESCE(f.filter1,0) + COALESCE(f.filter2,0) + COALESCE(f.filter3,0)) AS filter_count
  FROM userQS u
  LEFT JOIN PR_filters f 
         ON u.course_id = f.course_id AND u.question_id = f.question_id
),
```

## Step 4. Apply filters and compute question-level difficulty scores and user-level proficiency estimates.

```sql
qdiff AS (
  SELECT 
      course_id,
      question_id,
      AVG(correct)::decimal AS qdiff
  FROM userQS_with_filters
  WHERE filter_count = 0
  GROUP BY course_id, question_id
),

user_mean_qdiff AS (
    SELECT 
        course_id,
        user_id,
        AVG(q.qdiff)::decimal AS user_mean_qdiff
    FROM userQS_with_filters u
    LEFT JOIN qdiff q 
           ON u.course_id=q.course_id AND u.question_id=q.question_id
    GROUP BY course_id, user_id
)
```

## Step 5. Select final outputs for feeding the adaptive testing engine.
```sql
SELECT 
    'question_level_difficulty' AS result_type,
    course_id, 
    question_id, 
    qdiff AS difficulty_score
FROM qdiff

UNION ALL

SELECT 
    'user_level_difficulty',
    course_id,
    user_id AS entity_id,
    user_mean_qdiff AS difficulty_score
FROM user_mean_qdiff;
```


<!--

THIS IS PROBABLY TOO MUCH INFO FOR A SINGLE POST 

In addition to improving user experience through personalization, we also aimed to improve it through gamification. To that end, we wanted students to earn badges associated with predictors of success. Internal research suggested two key predictor of success were (1) study volume, and (2) study accuracy (specifically, stringing correct answers together in a row, which we call 'streaks').

3) Adding gamification features

When Problem Roulette was developed, its main value proposition was its test bank, which consisted of previously used test questions written by the professor of the course a student was taking. As we learned more about its ability to facilitate student success in the classroom, we wanted to find ways of encouraging its use. Personalization was one way we did so. Another way was through gamification. 

During the discovery process for building the proficiency score model, it was clear that "streaks" of correctly answered questions were a strong predictor of course grades. One interpretation is that streaks represent very intentional, effortful studying. It's tough to string together several correct responses. Doing so is rarely due to chance, so whatever a student did to earn a streak really worked well for them. Our goal was to signal this back to students in order to encourage reflection 

Figure 2. Mean course grade (GPE) by length of streak (number of questions completed in a row) and study volue (total number of questions completed).
<img src="/files/port-pr-streaks-plot.jpg" alt="PR Dimensions" style="max-width: 100%; height: auto;">

-->
