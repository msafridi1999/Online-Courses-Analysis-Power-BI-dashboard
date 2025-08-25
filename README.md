# Online-Courses-Analysis-Power-BI-dashboard
A complete, end-to-end Power BI project that analyzes recorded online courses collected from multiple EdTech platforms. The dashboard reveals which course types and skills are most popular, how language/subtitles affect viewership, which instructors consistently earn top ratings, and how course duration relates to audience engagement.


## Quick Links
- PBIX: `Online_Courses_Analysis.pbix`  
- Screenshots: `assets/Online_Course_Dashboard-1.png`  


## Business Problem (from the EdTech startup)
The startup wants to grow its library of recorded lectures. They have scraped/collected data from various EdTech sites and need clear insights to guide what to produce next and how to optimize accessibility.

Key questions (category-wise emphasis):
1. Distribution of course types across categories; count courses by category/sub-category to decide where to launch new content.
2. Average views for each category, sub-category, and language to understand engagement patterns.
3. Most commonly taught skills by category to keep offerings aligned to market demand.
4. Language distribution for courses within each category.
5. Language preference by category (analyze top 5 categories based on user preference).
6. Relationship between subtitles availability (count of subtitle languages) and views.
7. Top three instructors for each category/sub-category using ratings (static insight).
8. Relationship between course duration and views (normalize durations: 60 hours per month; flexible schedule = 200 hours).
9. Does skill variety within each category/sub-category impact viewership?

---

## Data Model (recommended star schema)
```
/data
  ├─ courses.csv                 # core facts (1 row per course)
  ├─ course_skills.csv           # bridge: many-to-many between courses and skills
  ├─ categories.csv              # dimCategory
  ├─ languages.csv               # dimLanguage
  ├─ instructors.csv             # dimInstructor
  └─ course_types.csv            # dimCourseType
```

Fact – `courses` (example columns):
- CourseID (key)
- Title
- Category, SubCategory, CourseType
- Language, SubtitleLanguageCount
- Views (viewer count / enrolments)
- DurationRaw, DurationUnit ("hours" | "months" | "flexible")
- InstructorName, InstructorRating
- SkillCount (optional helper if not using the bridge table)

Dimensions:
- categories(Category, SubCategory)
- languages(Language)
- instructors(InstructorName)
- course_types(CourseType)
- skills(Skill), course_skills(CourseID, Skill) for N:N

---

## Power Query — Core Transformations
1. Clean text: trim, clean, proper casing for Category, SubCategory, Language, CourseType.
2. Normalize Duration to Hours (new column):  
   - If DurationUnit = "hours" -> use the numeric value.  
   - If = "months" -> Hours = Months * 60 (assumption from problem statement).  
   - If = "flexible" -> Hours = 200 (assumption from problem statement).
3. Numeric casting: Views, SubtitleLanguageCount, InstructorRating, DurationHours -> numeric.
4. Skills split: split skills (comma/pipe) to rows -> output `course_skills` table (CourseID, Skill). Remove duplicates.
5. Create Date/Refresh columns if needed (optional).
6. Validate keys & relationships (1:* from dims to fact, and bridge table between courses and skills).

Keep the transformation steps descriptive (renamed columns, type changes) so other analysts can reproduce.

---

## Dashboard Pages & Visuals
All visuals shown in the screenshot are interactive with slicers for Category, Sub-Category, Course Type, and Language.

1. Course Type Popularity — bar chart of [Total Views] by CourseType (or Course Count depending on your definition).
2. Average Views by Sub-Category & Language — clustered columns: axis = SubCategory & Language, value = [Avg Views].
3. Most Common Skills by Category — bar of [Courses per Skill] (distinct count of courses linked to each skill) filtered by selected category.
4. Language Distribution — pie of [Course Count] by Language and a % of total measure.
5. Views vs Subtitles Count — line chart: axis = SubtitleLanguageCount, value = [Avg Views].
6. Instructor Rating (Top 3) — column chart of [Instructor Rating] by InstructorName; apply Top N = 3 (tie-breaker: [Avg Views]).
7. Views vs Duration (hrs) — line chart: axis = Duration (Hours) (or binned groups), value = [Avg Views].
8. Rank Category by Avg Views — table of Category and [Rank Category by Avg Views] (dense ranking).
9. Category summary table — Category with [Average Skill Count] and [Average Duration (Hours)].

---

## DAX Measures (copy/paste)
1. Rank_Category_By_Avg_Views = 
 IF(RANKX(ALL(Online_Courses[Category]),Calculate(Average(Online_Courses[Number of viewers])))<6,
        CALCULATE(AVERAGE(Online_Courses[Number of viewers])),
        BLANK())

 2. Instructor_Rating = IF(RANKX(ALL(Teachers[Instructors]),CALCULATE(AVERAGE(Teachers[Rating])))<=3,
                           CALCULATE(AVERAGE(Teachers[Rating])),
                           BLANK())

## Custom Column
Count of Skills Provided=Text.Length([Skills])-Text.Length(Text.Replace([Skills],
",",""))

## Custom Column
Duration in hours= if Text.Contains(Text.Lower([Duration]),"month") then
Number.FromText(Text.Select([Duration],{"0".."9"}))*60
else if Text.Contains(Text.Lower([Duration]),"hour") then
Number.FromText(Text.Select([Duration],{"0".."9"}))
else if Text.Contains(Text.Lower([Duration]),"minutes") then
Number.FromText(Text.Select([Duration],{"0".."9"}))/60
else 200
## Custom Column
3. custom=List.Count(Text.Split([Subtitle Languages],","))

## Interactions & Slicers
- Category / Sub-Category buttons at the top filter all visuals.
- Language slicer for comparing English vs other languages.
- Course Type slicer to isolate Courses, Specializations, Projects, and Professional Certificates.
- For the Top-3 Instructors visual, set the visual-level filter -> Top N = 3 by [Instructor Rating] and add [Avg Views] as a tie-breaker if needed.

## Insights Highlighted by the Sample Screenshot
- Course Type Popularity: "Course" dominates viewership versus Specializations/Projects/Certificates.
- Language Mix: English accounts for the bulk of courses; other languages form small but important niches.
- Skills Demand: Data Analysis, Python Programming, Data Science, and Machine Learning lead the skill count.
- Accessibility: Higher subtitle language counts correlate with spikes in average views at certain thresholds.
- Instructors: Consistently top-rated educators (e.g., Barbara Oakley, Beth Rogowsky, David Joyner, Dr. Terrence Sejnowski) can be prioritized for content partnerships.
- Duration vs Viewership: Distinct engagement peaks at certain hour ranges - useful for optimizing course length.

Final numbers will vary with your actual dataset. Use the measures here to reproduce the exact logic.

---

## Repository Structure
```
.
├─ assets/
│  ├─ Online_Course_Dashboard-1.png
│  └─ Project Problem Statement.jpg
├─ data/                    # (sample/raw/cleaned datasets)
├─ Online_Courses_Analysis.pbix
├─ README.md
└─ LICENSE
```

---

## How to Validate
- Cross-check totals (views, courses) between visuals by applying the same slicer selections.
- Use the Performance Analyzer to confirm measure/visual responsiveness.
- Optionally validate DAX with DAX Studio for row counts and filter context.

---

## Requirements
- Power BI Desktop (June 2023 or later recommended)
- (Optional) DAX Studio for diagnostics

## Author
Md Shahid Afridi (MS_AFRIDI)  
Data Analyst & Power BI Developer | Python • SQL • Power BI • ML  

---

## Acknowledgements
- Dataset aggregated from public EdTech sources (for learning/demo purposes).
- Inspiration: trends in online learning, language accessibility, and skills demand.
