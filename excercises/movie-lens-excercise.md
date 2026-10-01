```
reference notebook

https://github.com/training-sh/tcs-spark/blob/main/01_SparkDF/D304-MovieLens.ipynb

Break the logics of movilens.ipynb into 3 different notebooks

Notebook1: 001-Movies-Bronze-To-Silver.ipynb
            here read from hdfs, 
						   movies.csv from bronze

							 write to hdfs into a silver directory as parquet format

							 hdfs: /movielens/silver/movies/partXYZ....par

					  spark.stop


Notebook2: 002-Ratings-Bronze-To-Silver.ipynb


							 here read ratings.csv from hdfs, 
						   ratings.csv from bronze

							 write to hdfs into a silver directory as parquet format

							 hdfs: /movielens/silver/ratings/part-XYZ....par

					  spark.stop

Notebook3:
             read movies/partxyz.par files from silver into data frame
						 movie_df = spark.read.parquet(movies_silver_path) - no need to create schema, par has schema alreay
						 rating_df = spark.read.parquet(ratings_silver_path)

						 Find popular movies a movie rated by at least 100 user, rated avg at least 4.0 and above

						 result_df = join with movie_df to enrich data, you add title

						 Write the result_df into gold as parqeut, hdfs: /movielens/gold/popular-movies/part-XYZ....par

Notebook4:
        now you read popular movies from s3 gold zone [which is the output from notebook 3]
        Write to mysql , movielens database, popular_movies table

```
