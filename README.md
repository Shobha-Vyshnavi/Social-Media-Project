# Social-Media-Project
This project demonstrates how data analytics can transform social media user data into actionable business insights that drive growth and customer engagement.

This project focuses on analyzing Instagram user data to help the Marketing team at Meta develop data-driven marketing strategies that improve user engagement, retention, and acquisition.

Using data analytics techniques, the project identifies user behavior patterns, content preferences, engagement trends, and audience segmentation insights. The analysis supports strategic decision-making by helping marketers understand what drives user interaction and platform growth.

Project Objectives :
-Analyze Instagram user activity and engagement patterns
-Identify factors influencing user retention and acquisition
-Segment users based on behavior and interests
-Discover high-performing content trends
-Provide actionable marketing recommendations

Tools & Technologies:
SQL, Excel

Key Insights:
-User engagement trends across demographics
-Most effective posting times and content categories
-Retention analysis and churn indicators
-Influencer and campaign performance evaluation
-Audience targeting opportunities
-Business Impact

The insights generated from this project help marketing teams:
-Improve personalized marketing campaigns
-Increase Instagram user engagement
-Enhance customer retention strategies
-Optimize advertising performance
-Support data-driven business decisions



USE ig_clone;

## =================================================== OBJECTIVE QUESTIONS ==================================================


-- --------------------------------------------------------------------------------------------------------------------------
## Q1. Are there any tables with duplicate or missing null values? If so, how would you handle them?
-- --------------------------------------------------------------------------------------------------------------------------

-- CHECKING NULL VALUES 
SELECT * FROM comments 
WHERE 1 IS NULL OR 2 IS NULL OR 3 IS NULL OR 4 IS NULL OR 5 IS NULL ;

SELECT * FROM follows 
WHERE 1 IS NULL OR 2 IS NULL OR 3 IS NULL;

SELECT * FROM likes 
WHERE 1 IS NULL OR 2 IS NULL OR 3 IS NULL;

SELECT * FROM photo_tags
WHERE 1 IS NULL OR 2 IS NULL;

SELECT * FROM photos
WHERE 1 IS NULL OR 2 IS NULL OR 3 IS NULL OR 4 IS NULL;

SELECT * FROM tags
WHERE 1 IS NULL OR 2 IS NULL OR 3 IS NULL;

SELECT * FROM users
WHERE 1 IS NULL OR 2 IS NULL OR 3 IS NULL;

-- CHECKING DUPLICATE VALUES
 SELECT * FROM comments
 GROUP BY 1,2,3,4,5 HAVING COUNT(*) > 1;
 
  SELECT * FROM follows
 GROUP BY 1,2,3 HAVING COUNT(*) > 1;
 
  SELECT * FROM likes
 GROUP BY 1,2,3 HAVING COUNT(*) > 1;
 
  SELECT * FROM photo_tags
 GROUP BY 1,2 HAVING COUNT(*) > 1;
 
  SELECT * FROM photos
 GROUP BY 1,2,3,4 HAVING COUNT(*) > 1;
 
   SELECT * FROM tags
 GROUP BY 1,2,3 HAVING COUNT(*) > 1;
 
   SELECT * FROM users
 GROUP BY 1,2,3 HAVING COUNT(*) > 1;

-- --------------------------------------------------------------------------------------------------------------------------
## Q2. What is the distribution of user activity levels (e.g., number of posts, likes, comments) across the user base?
-- --------------------------------------------------------------------------------------------------------------------------
SELECT u.id AS user_id, u.username,
       COUNT(DISTINCT p.id) AS num_posts,
       COUNT(DISTINCT l.photo_id) AS num_likes,
       COUNT(DISTINCT c.id) AS num_comments
FROM users u
LEFT JOIN photos p ON u.id = p.user_id
LEFT JOIN likes l ON u.id = l.user_id
LEFT JOIN comments c ON u.id = c.user_id
GROUP BY u.id, u.username;

-- --------------------------------------------------------------------------------------------------------------------------
## Q3. Calculate the average number of tags per post (photo_tags and photos tables).
-- --------------------------------------------------------------------------------------------------------------------------

SELECT AVG(tags) AS avg_tags_per_post
FROM (
	SELECT p.id,COUNT(t.tag_id) AS tags
	FROM photos p
	LEFT JOIN photo_tags t ON p.id = t.photo_id 
	GROUP BY p.id
    ) AS num_tags;
-- --------------------------------------------------------------------------------------------------------------------------
## Q4. Identify the top users with the highest engagement rates (likes, comments) on their posts and rank them.
-- --------------------------------------------------------------------------------------------------------------------------

select u.id, u.username,
	coalesce(l.total_likes,0) as total_likes,
	coalesce(c.total_comments,0) as total_comments,
	(coalesce(l.total_likes, 0) + coalesce(c.total_comments, 0)) as total_engagements,
	dense_rank() over(order by (coalesce(l.total_likes,0) + coalesce(c.total_comments,0)) desc, u.id ) as engagement_rankings
from users u 
left join ( select user_id,count(*) as total_likes from likes group by user_id) l on u.id=l.user_id
left join ( select user_id, count(*) as total_comments from comments group by user_id) c on u.id=c.user_id
order by engagement_rankings;

-- --------------------------------------------------------------------------------------------------------------------------
## Q5. Which users have the highest number of followers and followings?
-- --------------------------------------------------------------------------------------------------------------------------

-- HIGHEST NO. OF FOLLOWERS
SELECT id,username,followers_count
FROM (
		SELECT id
		,username
		,COUNT(follower_id) AS followers_count
		,DENSE_RANK() OVER(ORDER BY COUNT(follower_id) DESC) AS count_rank
		FROM users u 
		LEFT JOIN follows f ON u.id=f.followee_id 
		GROUP BY 1,2
) Z
WHERE count_rank=1
ORDER BY 1;

-- HIGHEST NO. OF FOLLOWINGS
SELECT id,username,followings_count
FROM (
		SELECT id
		,username
		,COUNT(followee_id) AS followings_count
		,DENSE_RANK() OVER(ORDER BY COUNT(followee_id) DESC) AS count_rank
		FROM users u 
		LEFT JOIN follows f ON u.id=f.follower_id 
		GROUP BY 1,2
) Z
WHERE count_rank=1
ORDER BY 1;
-- --------------------------------------------------------------------------------------------------------------------------
## Q6. Calculate the average engagement rate (likes, comments) per post for each user.
-- --------------------------------------------------------------------------------------------------------------------------

SELECT 
	u.id as user_id,
	u.username,
	COALESCE(p.num_posts, 0) AS num_posts,
	COALESCE(l.num_likes, 0) AS num_likes,
	COALESCE(c.num_comments, 0) AS num_comments,
	CASE 
		WHEN COALESCE(p.num_posts, 0) = 0 THEN 0
		ELSE (COALESCE(l.num_likes, 0) + COALESCE(c.num_comments, 0)) / COALESCE(p.num_posts, 0)
	END AS avg_engagement_rate
FROM users u
LEFT JOIN (SELECT user_id, COUNT(*) AS num_posts FROM photos
     GROUP BY user_id) p ON u.id = p.user_id
LEFT JOIN (SELECT user_id, COUNT(*) AS num_likes
     FROM likes
     GROUP BY user_id) l ON u.id = l.user_id
LEFT JOIN (SELECT user_id, COUNT(*) AS num_comments FROM comments GROUP BY user_id) c ON u.id = c.user_id
	ORDER BY avg_engagement_rate DESC;

-- --------------------------------------------------------------------------------------------------------------------------    
## Q7. Get the list of users who have never liked any post (users and likes tables)
-- --------------------------------------------------------------------------------------------------------------------------

SELECT id, username
FROM users 
WHERE id IN (
	SELECT id FROM users
	EXCEPT
	SELECT user_id FROM likes
    );

-- --------------------------------------------------------------------------------------------------------------------------    
## Q8. How can you leverage user-generated content (posts, hashtags, photo tags) to create more personalized and engaging ad 
-- campaigns?
-- --------------------------------------------------------------------------------------------------------------------------

-- Finding out tag_name for each photo liked by user
WITH tag_name AS(
SELECT l.user_id,tag_name,l.photo_id
FROM likes l
LEFT JOIN photo_tags pt ON  pt.photo_id=l.photo_id
LEFT JOIN tags t ON pt.tag_id=t.id
),

-- giving a tag_category to each tag_name to better understand the user liking
tag_category AS(
SELECT id tag_id,tag_name
, CASE WHEN tag_name IN ('happy', 'smile') THEN 'Joy-Emotions'
    WHEN tag_name IN ('stunning', 'dreamy') THEN 'Aesthetics'
    WHEN tag_name IN ('delicious', 'food', 'foodie') THEN 'Food'
    WHEN tag_name IN ('concert', 'party', 'drunk', 'lol','fun') THEN 'Party & Fun'
    WHEN tag_name IN ('beauty', 'hair') THEN 'Beauty'
    WHEN tag_name IN ('landscape', 'sunrise', 'sunset','beach') THEN 'Landscape'
    WHEN tag_name IN ('fashion', 'style') THEN 'Fashion'
    WHEN tag_name = 'photography' THEN 'Photography'
    ELSE NULL
END AS tag_category
FROM tags
),

-- counting the no. of likes a user did in each category and then ranking it 
likes_per_category AS (
SELECT user_id
,tag_category
,COUNT(photo_id) AS likes_done
,DENSE_RANK() OVER(PARTITION BY user_id ORDER BY COUNT(photo_id) DESC) AS likes_rank
FROM tag_name tn 
JOIN tag_category tc ON tn.tag_name=tc.tag_name
GROUP BY 1,2
ORDER BY 1,3 DESC
)

SELECT user_id,tag_category,likes_done
FROM likes_per_category a
WHERE likes_rank<=3;


-- --------------------------------------------------------------------------------------------------------------------------    
## Q9. Are there any correlations between user activity levels and specific content types (e.g., photos, videos, reels)? How 
--     can this information guide content creation and curation strategies?
-- --------------------------------------------------------------------------------------------------------------------------

-- User Upload Activity:
WITH uploads AS (
SELECT u.id user_id,
    u.username, 
    COUNT(p.id) AS photo_uploads
FROM users u
LEFT JOIN photos p ON u.id = p.user_id
GROUP BY 1,2
),

-- Likes per User's Photos:
likes AS (
SELECT u.id user_id
    , u.username 
    , COUNT(l.photo_id) AS total_likes
FROM users u
LEFT JOIN photos p ON u.id = p.user_id
LEFT JOIN likes l ON p.id = l.photo_id
GROUP BY 1,2
),

-- comments per User's Photos:
comments AS(
SELECT u.id user_id
    , u.username
    , COUNT(c.id) AS total_comments
FROM users u
LEFT JOIN photos p ON u.id = p.user_id
LEFT JOIN comments c ON p.id = c.photo_id
GROUP BY 1,2
)

-- CORRELATION BETWEEN NO. OF UPLOADS AND TOTAL_ENGAGAEMNET
	SELECT DISTINCT photo_uploads
	,ROUND(AVG(total_likes+total_comments) OVER(PARTITION BY photo_uploads),0) average_engagement
	FROM uploads u 
	JOIN likes l ON u.user_id=l.user_id
	JOIN comments c ON u.user_id=c.user_id;

-- --------------------------------------------------------------------------------------------------------------------------
## Q10. Calculate the total number of likes, comments, and photo tags for each user.
-- --------------------------------------------------------------------------------------------------------------------------

SELECT 
	u.id as id, 
    u.username,
	COALESCE(l.total_likes, 0) AS total_likes,
	COALESCE(c.total_comments, 0) AS total_comments,
	COALESCE(pt.total_photo_tags, 0) AS total_photo_tags
FROM users u
LEFT JOIN (
	SELECT 
		user_id, 
		COUNT(*) AS total_likes 
    FROM likes 
    GROUP BY user_id
) l ON u.id = l.user_id
LEFT JOIN (
	SELECT 
		user_id, 
        COUNT(*) AS total_comments 
        FROM comments 
        GROUP BY user_id
) c ON u.id = c.user_id
LEFT JOIN (
	SELECT 
		tag_id, 
        COUNT(*) AS total_photo_tags 
        FROM photo_tags 
        GROUP BY tag_id
) pt ON u.id = pt.tag_id;
	
-- --------------------------------------------------------------------------------------------------------------------------
## Q11. Rank users based on their total engagement (likes, comments, shares) over a month.
-- --------------------------------------------------------------------------------------------------------------------------

WITH engagement AS (
SELECT u.id AS user_id
,username
,MONTH(p.created_dat) `month`
,YEAR(p.created_dat) `year`
,p.id AS photo_id
,(COUNT(DISTINCT c.user_id)+ COUNT(DISTINCT l.user_id)) AS engagement_recieved
FROM users u 
LEFT JOIN photos p ON u.id=p.user_id
LEFT JOIN comments c ON c.photo_id=p.id
LEFT JOIN likes l ON l.photo_id=p.id
GROUP BY 1,2,3,4,5
)
SELECT user_id, username,`year`,`month`,SUM(engagement_recieved) AS total_engagement
,DENSE_RANK() OVER(PARTITION BY year, month ORDER BY SUM(engagement_recieved) DESC) AS engagement_rank
FROM engagement
where month is not null
GROUP BY 1,2,3,4;

-- --------------------------------------------------------------------------------------------------------------------------
## Q12. Retrieve the hashtags that have been used in posts with the highest average number of likes.
-- 		Use a CTE to calculate the average likes for each hashtag first.
-- --------------------------------------------------------------------------------------------------------------------------

WITH tag_name_and_likes AS (
SELECT t.id tag_id
,tag_name
,pt.photo_id 
,COUNT( DISTINCT l.user_id) total_likes
,AVG(COUNT( DISTINCT l.user_id)) OVER(PARTITION BY t.id) AS avg_likes
FROM tags t 
LEFT JOIN photo_tags pt ON t.id=pt.tag_id
JOIN likes l ON l.photo_id=pt.photo_id
GROUP BY 1,2,3
)

SELECT DISTINCT tag_id,tag_name
FROM tag_name_and_likes
WHERE avg_likes IN (SELECT MAX(avg_likes) FROM tag_name_and_likes )
ORDER BY 1;

-- --------------------------------------------------------------------------------------------------------------------------
## Q13. Retrieve the users who have started following someone after being followed by that person
-- --------------------------------------------------------------------------------------------------------------------------

SELECT 
	f1.follower_id, 
    f1.followee_id
FROM follows f1
JOIN follows f2 ON f1.follower_id = f2.followee_id AND f1.followee_id = f2.follower_id
WHERE f1.created_at > f2.created_at;

---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

use ig_clone;

## =================================================== SUBJECTIVE QUESTIONS =================================================

-- --------------------------------------------------------------------------------------------------------------------------
## Q1. Based on user engagement and activity levels, which users would you consider the most loyal or valuable? 
-- 	   How would you reward or incentivize these users?
-- --------------------------------------------------------------------------------------------------------------------------

WITH TotalLikes AS (
    SELECT u.id, COUNT(distinct l.photo_id) AS total_likes
    FROM users u
    LEFT JOIN likes l ON u.id = l.user_id
    GROUP BY u.id),
TotalComments AS (
    SELECT u.id, COUNT(distinct c.photo_id) AS total_comments
    FROM users u
    LEFT JOIN comments c ON u.id = c.user_id
    GROUP BY u.id),
PhotosPosted AS (
    SELECT user_id, COUNT(id) AS total_photos_posted
    FROM photos
    GROUP BY user_id),
Followers AS (
    SELECT followee_id AS user_id, COUNT(follower_id) AS total_followers
    FROM follows
    GROUP BY followee_id),
UniqueTags AS (
    SELECT p.user_id, COUNT(DISTINCT pt.tag_id) AS unique_tags_used
    FROM photos p
    LEFT JOIN photo_tags pt ON p.id = pt.photo_id
    GROUP BY p.user_id)
SELECT u.id AS user_id, u.username,
    COALESCE(tl.total_likes, 0) AS total_likes,
    COALESCE(tc.total_comments, 0) AS total_comments,
    COALESCE(pp.total_photos_posted, 0) AS total_photos_posted,
    COALESCE(f.total_followers, 0) AS total_followers,
    COALESCE(ut.unique_tags_used, 0) AS unique_tags_used,
    (COALESCE(tl.total_likes, 0) + COALESCE(tc.total_comments, 0)) AS total_engagement
FROM users u
LEFT JOIN TotalLikes tl ON u.id = tl.id
LEFT JOIN TotalComments tc ON u.id = tc.id
LEFT JOIN PhotosPosted pp ON u.id = pp.user_id
LEFT JOIN Followers f ON u.id = f.user_id
LEFT JOIN UniqueTags ut ON u.id = ut.user_id
group by u.id 
having total_photos_posted >0
ORDER BY total_engagement DESC, total_followers DESC, total_photos_posted DESC
limit 10;

-- --------------------------------------------------------------------------------------------------------------------------    
## Q2. For inactive users, what strategies would you recommend to re-engage them and encourage them to start posting or 
--     engaging again?
-- --------------------------------------------------------------------------------------------------------------------------

WITH likes_count AS (
	SELECT
		DISTINCT user_id,
		count(*) AS num_of_likes
	FROM likes
	GROUP BY User_id
),
comments_count AS (
	SELECT
		user_id,
		count(id) AS num_of_comments
	FROM comments
	GROUP BY user_id
),
photo_counts AS (
	SELECT
		user_id,
		COUNT(*) AS num_of_photos
	FROM photos
	GROUP BY user_id
),
phototags_count AS (
	SELECT
		p.user_id,
		count(pt.tag_id) AS num_of_phototags
	FROM photos p
	JOIN photo_tags AS pt ON p.user_id = pt.photo_id
	GROUP BY p.user_id
),
Count_of_followers AS (
	SELECT
		follower_id,
		count(follower_id) AS follower_count,
		count(followee_id) AS followee_count
	FROM follows
	GROUP BY follower_id
)
SELECT
	u.id AS UserID,
	u.username AS UserName,
	coalesce(l.num_of_likes, 0) AS num_of_likes,
	coalesce(c.num_of_comments, 0) AS num_of_comments,
	coalesce(pp.num_of_photos, 0) AS num_of_photos,
	coalesce(p.num_of_phototags, 0) AS num_of_phototags,
	coalesce(f.follower_count, 0) AS follower_count,
	coalesce(f.followee_count, 0) AS followee_count,
	coalesce(
		(coalesce(l.num_of_likes, 0) + coalesce(c.num_of_comments, 0) + coalesce(pp.num_of_photos, 0)),0
	) AS engagement_rate,
	DENSE_RANK() OVER (
		ORDER BY
			(coalesce(l.num_of_likes, 0) + coalesce(c.num_of_comments, 0) + coalesce(pp.num_of_photos, 0)) ASC
	) AS engagement_rate_rank
FROM
	users u
	LEFT JOIN likes_count AS l ON u.id = l.user_id
	LEFT JOIN comments_count AS c ON u.id = c.user_id
	LEFT JOIN photo_counts AS pp ON u.id = pp.user_id
	LEFT JOIN phototags_count AS p ON u.id = p.user_id
	LEFT JOIN Count_of_followers AS f ON u.id = f.follower_id
ORDER BY
	engagement_rate_rank ASC;

-- --------------------------------------------------------------------------------------------------------------------------
## Q3. Which hashtags or content topics have the highest engagement rates? 
-- 	   How can this information guide content strategy and ad campaigns?
-- --------------------------------------------------------------------------------------------------------------------------

WITH PhotoEngagement AS (
    SELECT
        p.id AS photo_id,
        COUNT(distinct l.photo_id) AS total_likes,
        COUNT(DISTINCT c.id) AS total_comments,
        COUNT(distinct l.photo_id) + COUNT(DISTINCT c.user_id) AS total_engagement
    FROM photos p
    LEFT JOIN likes l ON p.user_id = l.user_id
    LEFT JOIN comments c ON p.user_id = c.user_id
    GROUP BY p.id),
HashtagEngagement AS (
    SELECT
        t.id AS tag_id,
        t.tag_name,
        count(pe.total_engagement) AS total_engagement,
        COUNT(DISTINCT pt.photo_id) AS total_photos,
        (count(pe.total_engagement) / COUNT(DISTINCT pt.photo_id) )AS engagement_rate
    FROM tags t
    JOIN photo_tags pt ON t.id = pt.tag_id
    JOIN PhotoEngagement pe ON pt.photo_id = pe.photo_id
    GROUP BY t.id, t.tag_name)
SELECT tag_name, total_photos, total_engagement, engagement_rate
FROM HashtagEngagement
ORDER BY total_engagement DESC
limit 10;

-- --------------------------------------------------------------------------------------------------------------------------
## Q4. Are there any patterns or trends in user engagement based on demographics (age, location, gender) 
-- 	   or posting times? How can these insights inform targeted marketing campaigns?
-- --------------------------------------------------------------------------------------------------------------------------

SELECT 
    HOUR(p.created_dat) AS post_hour,
    DAYOFWEEK(p.created_dat) AS post_day,
    COUNT(DISTINCT p.id) AS total_photos_posted,
    COUNT(DISTINCT l.photo_id) AS total_likes_received,
    COUNT(DISTINCT c.id) AS total_comments_made
FROM photos p
 JOIN likes l ON p.id = l.photo_id
 JOIN comments c ON p.id = c.photo_id
GROUP BY post_hour, post_day
ORDER BY post_hour, post_day;

-- --------------------------------------------------------------------------------------------------------------------------
## Q5. Based on follower counts and engagement rates, which users would be ideal candidates for influencer marketing campaigns? 
-- 	   How would you approach and collaborate with these influencers?
-- --------------------------------------------------------------------------------------------------------------------------

WITH TotalLikes AS (
    SELECT u.id, COUNT(distinct l.photo_id) AS total_likes
    FROM users u
    LEFT JOIN likes l ON u.id = l.user_id
    GROUP BY u.id),
TotalComments AS (
    SELECT u.id, COUNT(distinct c.photo_id) AS total_comments
    FROM users u
    LEFT JOIN comments c ON u.id = c.user_id
    GROUP BY u.id),
PhotosPosted AS (
    SELECT user_id, COUNT(id) AS total_photos_posted
    FROM photos
    GROUP BY user_id),
Followers AS (
    SELECT followee_id AS user_id, COUNT(follower_id) AS total_followers
    FROM follows
    GROUP BY followee_id)
SELECT u.id AS user_id, u.username,
    COALESCE(tl.total_likes, 0) AS total_likes,
    COALESCE(tc.total_comments, 0) AS total_comments,
    COALESCE(pp.total_photos_posted, 0) AS total_photos_posted,
    COALESCE(f.total_followers, 0) AS total_followers,
    (
		(COALESCE(tl.total_likes, 0) + COALESCE(tc.total_comments, 0))
        /
        (COALESCE(pp.total_photos_posted, 0))
    )  as engagement_rate
FROM users u
JOIN TotalLikes tl ON u.id = tl.id
JOIN TotalComments tc ON u.id = tc.id
JOIN PhotosPosted pp ON u.id = pp.user_id
JOIN Followers f ON u.id = f.user_id
group by u.id 
having total_photos_posted >0
ORDER BY  engagement_rate desc, total_followers desc,total_photos_posted desc 
limit 10;

-- --------------------------------------------------------------------------------------------------------------------------
## Q6. Based on user behavior and engagement data, 
-- 	   how would you segment the user base for targeted marketing campaigns or personalized recommendations?
-- --------------------------------------------------------------------------------------------------------------------------

SELECT 
    u.id AS user_id,
    u.username,
    COALESCE(p.total_posts, 0) AS total_posts,
    COALESCE(l.total_likes, 0) + COALESCE(c.total_comments, 0) AS total_engagement,
    CASE
		WHEN COALESCE(l.total_likes, 0) + COALESCE(c.total_comments, 0) > 150
			THEN 'Highly Engaged'
		WHEN COALESCE(l.total_likes, 0) + COALESCE(c.total_comments, 0) BETWEEN 100 AND 150 
			THEN 'Moderately Engaged'
		ELSE 'Less Engaged' END AS user_category,
    CASE 
	WHEN YEAR(u.created_at) >= 2017 THEN 'New_User'
	ELSE 'Old_User' END AS user_join_status
FROM users u
LEFT JOIN (
    SELECT user_id, COUNT(*) AS total_likes
    FROM ig_clone.likes
    GROUP BY user_id) l ON u.id = l.user_id
LEFT JOIN (
    SELECT user_id, COUNT(*) AS total_comments
    FROM comments
    GROUP BY user_id) c ON u.id = c.user_id
LEFT JOIN (
    SELECT user_id, COUNT(*) AS total_posts
    FROM photos
    GROUP BY user_id) p ON u.id = p.user_id
GROUP BY u.id
HAVING total_posts > 0 
ORDER BY total_engagement DESC, user_category;

-- --------------------------------------------------------------------------------------------------------------------------    
## Q7. If data on ad campaigns (impressions, clicks, conversions) is available, how would you measure their effectiveness 
--     and optimize future campaigns?
-- --------------------------------------------------------------------------------------------------------------------------

-- ****** Answered in Documented pdf file ******

-- --------------------------------------------------------------------------------------------------------------------------    
## Q8. How can you use user activity data to identify potential brand ambassadors or advocates who could help promote 
--     Instagram's initiatives or events?
-- --------------------------------------------------------------------------------------------------------------------------

-- ****** Answered in Documented pdf file ******

-- --------------------------------------------------------------------------------------------------------------------------    
## Q9. How would you approach this problem, if the objective and subjective questions weren't given?
-- --------------------------------------------------------------------------------------------------------------------------

-- ****** Answered in Documented pdf file ******

-- --------------------------------------------------------------------------------------------------------------------------
## Q10. Assuming there's a "User_Interactions" table tracking user engagements, how can you update the "Engagement_Type" 
--      column to change all instances of "Like" to "Heart" to align with Instagram's terminology?
-- --------------------------------------------------------------------------------------------------------------------------

CREATE TABLE User_Interactions(
	id INT AUTO_INCREMENT UNIQUE PRIMARY KEY,
	username VARCHAR(250) NOT NULL,
	Engagement_Type varchar(250) not null
);

INSERT INTO User_Interactions (username,Engagement_Type ) VALUES 
('Kenton_Kirlin', 'Like'),
 ('Andre_Purdy85', 'Comments'), 
 ('Harley_Lind18', 'Comments'),
 ('Arely_Bogan63', 'Comments'), 
 ('Aniya_Hackett', 'Like'), 
 ('Travon.Waters', 'Like'), 
 ('Kasandra_Homenick', 'Comments'), 
 ('Tabitha_Schamberger11', 'Comments'),
 ('Gus93', 'Like'), 
 ('Presley_McClure', 'Comments'), 
 ('Justina.Gaylord27', 'Like'), 
 ('Dereck65', 'Comments'), 
 ('Alexandro35', 'Comments'), 
 ('Jaclyn81', 'Comments'), 
 ('Billy52', 'Like'), 
 ('Annalise.McKenzie16', 'Comments'), 
 ('Norbert_Carroll35', 'Like'), 
 ('Odessa2', 'Comments'), 
 ('Hailee26', 'Comments'), 
 ('Delpha.Kihn', 'Like'), 
 ('Rocio33', 'Like'), 
 ('Kenneth64', 'Like'), 
 ('Eveline95', 'Like'),
 ('Maxwell.Halvorson', 'Like'), 
 ('Tierra.Trantow', 'Like'),
 ('Josianne.Friesen', 'Like'), 
 ('Darwin29', 'Like'), 
 ('Dario77', 'Like'),
 ('Jaime53', 'Comments'),
 ('Kaley9', 'Comments'), 
 ('Aiyana_Hoeger', 'Like'), 
 ('Irwin.Larson', 'Like'), 
 ('Yvette.Gottlieb91', 'Comments'), 
 ('Pearl7', 'Like'), 
 ('Lennie_Hartmann40', 'Comments'), 
 ('Ollie_Ledner37', 'Like'), 
 ('Yazmin_Mills95', 'Comments'), 
 ('Jordyn.Jacobson2', 'Like'), 
 ('Kelsi26', 'Like'), 
 ('Rafael.Hickle2', 'Comments'), 
 ('Mckenna17', 'Like'), 
 ('Maya.Farrell', 'Comments'), 
 ('Janet.Armstrong', 'Like'), 
 ('Seth46', 'Comments'), 
 ('David.Osinski47', 'Like'), 
 ('Malinda_Streich', 'Comments'), 
 ('Harrison.Beatty50', 'Like'), 
 ('Granville_Kutch', 'Comments'), 
 ('Morgan.Kassulke', 'Like'), 
 ('Gerard79', 'Comments'), 
 ('Mariano_Koch3', 'Comments'), 
 ('Zack_Kemmer93', 'Like'), 
 ('Linnea59', 'Comments'), 
 ('Duane60', 'Comments'), 
 ('Meggie_Doyle', 'Like'), 
 ('Peter.Stehr0', 'Comments'), 
 ('Julien_Schmidt', 'Like'), 
 ('Aurelie71', 'Comments'), 
 ('Cesar93', 'Comments'), 
 ('Sam52', 'Like'), 
 ('Jayson65', 'Comments'), 
 ('Ressie_Stanton46', 'Like'), 
 ('Elenor88', 'Comments'), 
 ('Florence99', 'Like'), 
 ('Adelle96', 'Comments'), 
 ('Mike.Auer39', 'Comments'), 
 ('Emilio_Bernier52', 'Like'), 
 ('Franco_Keebler64', 'Comments'), 
 ('Karley_Bosco', 'Like'), 
 ('Erick5', 'Comments'), 
 ('Nia_Haag', 'Like'), 
 ('Kathryn80', 'Comments'), 
 ('Jaylan.Lakin', 'Like'), 
 ('Hulda.Macejkovic', 'Comments'), 
 ('Leslie67', 'Comments'), 
 ('Janelle.Nikolaus81', 'Like'), 
 ('Donald.Fritsch', 'Comments'), 
 ('Colten.Harris76', 'Like'), 
 ('Katarina.Dibbert', 'Comments'), 
 ('Darby_Herzog', 'Comments'), 
 ('Esther.Zulauf61', 'Like'), 
 ('Aracely.Johnston98', 'Comments'), 
 ('Bartholome.Bernhard', 'Comments'), 
 ('Alysa22', 'Comments'), 
 ('Milford_Gleichner42', 'Like'), 
 ('Delfina_VonRueden68', 'Comments'), 
 ('Rick29', 'Like'), 
 ('Clint27', 'Comments'), 
 ('Jessyca_West', 'Comments'), 
 ('Esmeralda.Mraz57', 'Like'), 
 ('Bethany20', 'Comments'), 
 ('Frederik_Rice', 'Comments'), 
 ('Willie_Leuschke', 'Like'), 
 ('Damon35', 'Comments'), 
 ('Nicole71', 'Comments'), 
 ('Keenan.Schamberger60', 'Like'), 
 ('Tomas.Beatty93', 'Comments'), 
 ('Imani_Nicolas17', 'Like'), 
 ('Alek_Watsica', 'Comments'), 
 ('Javonte83', 'Like');
 
 SET SQL_SAFE_UPDATES = 0;
 update  User_Interactions 
 set  Engagement_Type = "Heart" 
 where Engagement_Type= "Like";
 
select * from User_Interactions; 
 
 
