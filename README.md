# Online-Courses-Analysis-Power-BI-dashboard
A complete, end-to-end Power BI project that analyzes recorded online courses collected from multiple EdTech platforms. The dashboard reveals which course types and skills are most popular, how language/subtitles affect viewership, which instructors consistently earn top ratings, and how course duration relates to audience engagement.

<p align="center">
  <img src="assets/Online_Course_Dashboard-1.png" alt="Online Courses Analysis Dashboard" width="85%"/>
</p>

---

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
Note: Rename table/column names to match your model. These measures are designed to work with a star schema (facts + dims + bridge).

```DAX
-- BASIC COUNTS
Total Courses =
DISTINCTCOUNT ( courses[CourseID] )

Total Instructors =
DISTINCTCOUNT ( instructors[InstructorName] )

Course Count =
COUNTROWS ( courses )

-- VIEWS
Total Views =
SUM ( courses[Views] )

Avg Views =
AVERAGE ( courses[Views] )

Avg Views (Per Course) =
DIVIDE ( [Total Views], [Total Courses] )

-- DURATION (HOURS) - CALCULATED COLUMN (in 'courses')
Duration (Hours) =
VAR unit = courses[DurationUnit]
VAR raw  = VALUE ( courses[DurationRaw] )
RETURN
    SWITCH (
        TRUE (),
        unit = "hours",     raw,
        unit = "months",    raw * 60,
        unit = "flexible",  200,
        BLANK ()
    )

-- AVERAGES FOR TABLES
Average Duration (Hours) =
AVERAGE ( courses[Duration (Hours)] )

Average Skill Count =
AVERAGE ( courses[SkillCount] )

-- LANGUAGE SHARE
Language Course Count =
CALCULATE ( [Total Courses], ALLSELECTED ( languages[Language] ) )

Language Share % =
DIVIDE (
    [Total Courses],
    CALCULATE ( [Total Courses], ALL ( languages[Language] ) ),
    0
)

-- SKILLS: assumes a bridge table `course_skills (CourseID, Skill)`
Courses per Skill =
DISTINCTCOUNT ( course_skills[CourseID] )

-- RANKING
Rank Category by Avg Views =
RANKX (
    ALL ( categories[Category] ),
    CALCULATE ( [Avg Views] ),
    ,
    DESC,
    DENSE
)

-- INSTRUCTORS
Instructor Rating =
AVERAGE ( courses[InstructorRating] )

-- For a Top N visual: use the Filter pane -> Top N = 3 by [Instructor Rating]
-- Optional tie-breaker rank for display:
Instructor Rank (by Rating, then Views) =
RANKX (
    ALLSELECTED ( instructors[InstructorName] ),
    [Instructor Rating] * 1000000 + [Avg Views],
    ,
    DESC,
    DENSE
)

-- SUBTITLES & VIEWS
Avg Views by Subtitles Count =
AVERAGEX (
    VALUES ( courses[SubtitleLanguageCount] ),
    CALCULATE ( [Avg Views] )
)

-- TOP 5 CATEGORIES (FOR LANGUAGE PREFERENCE ANALYSIS)
Is Top 5 Category (by Views) =
VAR _top =
    TOPN ( 5, VALUES ( categories[Category] ), [Total Views], DESC )
RETURN
    IF (
        CONTAINS ( _top, categories[Category], SELECTEDVALUE ( categories[Category] ) ),
        1,
        0
    )
```

Suggested bins for Duration chart: create a Grouping on Duration (Hours) (e.g., 0-20, 21-40, ..., or 0-100, 101-200, ... depending on your distribution).

---

## Interactions & Slicers
- Category / Sub-Category buttons at the top filter all visuals.
- Language slicer for comparing English vs other languages.
- Course Type slicer to isolate Courses, Specializations, Projects, and Professional Certificates.
- For the Top-3 Instructors visual, set the visual-level filter -> Top N = 3 by [Instructor Rating] and add [Avg Views] as a tie-breaker if needed.

---

## How to Reproduce (Step-by-Step)
1. Clone this repo and open `Online_Courses_Analysis.pbix` in Power BI Desktop (2023+).
2. Put the CSVs/Excel under `data/` and update the Data Source Settings if prompted.
3. In Power Query, apply the transformations listed above, especially the Duration (Hours) normalization.
4. Create relationships:
   - categories[Category] -> courses[Category] (1:*)
   - categories[SubCategory] -> courses[SubCategory] (1:*)
   - languages[Language] -> courses[Language] (1:*)
   - instructors[InstructorName] -> courses[InstructorName] (1:*)
   - courses[CourseID] <-> course_skills[CourseID] (1:*), and skills[Skill] <-> course_skills[Skill] (1:*)
5. Create the DAX measures above.
6. Rebuild visuals following the Dashboard Pages & Visuals section (use the screenshot as your guide).
7. Save and refresh. Publish to Power BI Service if needed.

---

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

---

## License
MIT — feel free to use and adapt with attribution.

---

## Author
Md Shahid Afridi (MS_AFRIDI)  
Data Scientist & ML Engineer | Python • SQL • Power BI • ML  
Portfolio & projects: (add your links here)

---

## Acknowledgements
- Dataset aggregated from public EdTech sources (for learning/demo purposes).
- Inspiration: trends in online learning, language accessibility, and skills demand.
