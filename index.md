
This is a quick guide to extract the mongoose GPS data using GPSBabel. I've also provided some code for the first few data cleaning steps. The rest is available - just need to get around to adding it to here!

# Using GPSBabel to convert the map files

First, copy and paste all the .gdb files from each group's map folder into a single new folder somewhere on your computer. I ended up with around 8000 files in this new folder.

We now want to convert these files into a format that other software, like R, can read. To do this, we can use a very useful programme called GPSBabel, developed by Robert Lipe. Download it from here: [GPSBabel](https://www.gpsbabel.org). Note, if you ever need help with something GPSBabel related, you can email the mailing list, found on the [GPSBabel help page](https://www.gpsbabel.org//lists.html) - the creator was super helpful when I emailed with a problem.
\
\
Make a note of where it is installed on your computer. For me, the default is "C:/Program Files (x86)/GPSBabel". Once you've finished installing it, we want to create a file in Windows that will batch process all of our .gdb files for us automatically. To do this, navigate to the new folder you created which stores all the .gdb files. Right click somewhere, select "New -> Text document", and open this text file.

We then want to paste in the below code:
```{bash, eval = FALSE}
Setlocal enabledelayedexpansion
Set "Pattern= "
Set "Replace="

For %%a in (*.gdb) Do (
    Set "File=%%~a"
    Ren "%%a" "!File:%Pattern%=%Replace%!"
)

For %%i in (*.gdb) do "C:\Program Files (x86)\GPSBabel\GPSBabel" -w -i gdb -f %%i -x nuketypes,routes,tracks -o unicsv -F %%~Ni_converted.unicsv
for /f %%a in ('dir /b *.unicsv') do for /f "tokens=*" %%b in (%%a) do echo %%b,%%a >> merged_waypoints.unicsv
del *_converted.unicsv
pause
```
\
Note that this section points to where you have GPSBabel installed on your computer:
```{bash, eval = FALSE}
 "C:\Program Files (x86)\GPSBabel\GPSBabel"
```
You must change this to wherever you previously installed GPSBabel. If you can't remember, in the taskbar, search for "GPSBabel", right click it, select "Open file location". If this takes you to a shortcut, right click the application and select "Open file location" again. It's likely installed somewhere like "C:\\Program Files\\GPSBabel".  
\
The below section removes any spaces in the file name, otherwise the file is not found and skipped during the file conversion.
```{bash, eval = FALSE}
Setlocal enabledelayedexpansion
Set "Pattern= "
Set "Replace="

For %%a in (*.gdb) Do (
    Set "File=%%~a"
    Ren "%%a" "!File:%Pattern%=%Replace%!"
)
```
\
This section:
```{bash, eval = FALSE}
`-w -i gdb -f %%i -x nuketypes,routes,tracks -o unicsv -F %%~Ni_converted.unicsv
```
tells GPSBabel to extract waypoints `-w` from the gdb files and delete routes and tracks `-x nuketypes,routes,tracks`. If you wanted just tracks, you could use `-t` instead of `-w` at the start, and use `-x nuketypes,waypoints,routes`. The code `-F %%~Ni_converted.unicsv` renames each .gdb output file, adding "_converted.unicsv" onto the end of each file. The unicsv extension allows for better conversion of the .gdb files than just .csv.
\
\
This part:
```{bash, eval = FALSE}
for /f %%a in ('dir /b *.unicsv') do for /f "tokens=*" %%b in (%%a) do echo %%b,%%a >> merged_waypoints.unicsv`
```
takes each individual file we have converted, and merges them all into one big unicsv file called "merged_waypoints". It also adds an additional column to the end of the data, which stores the file name of each converted file. That way we can check for duplicates within each file, and we can find any .gdb files that may have errors/outliers in them later on. If you want to now delete all the individually converted files, you can use `del *_converted.unicsv` at the end of the script, which deletes all files that have "_converted.unicsv" in them.
\
Note, with .gdb files, GPSBabel also extracts waypoints from the routes of the GPS, which is why we are deleting the routes. However, lots of duplicate waypoints still end up in our final file, which we clean later in R.
\
\
Once you've pasted the text in to the text document, save the file with "Save as", and change the drop down box "Save as type" from "Text document" to the "All files" option. Then name the file something, making sure it ends with ".bat". For example, "get_waypoints.bat". This .bat extension saves the file as a batch file. This is now able to convert all the .gdb files in the same folder as the batch file, into a merged .unicsv file. WARNING: before you run this file to convert the .gdb files, I'd recommend testing this out with a few map files in a separate folder, just in case anything goes wrong. You should be able to just double click the file to open it, and it'll start running - depending on how many files you have it may take a while.
\

# Importing and cleaning the data in R

\
Now we've got the waypoints in a unicsv file, we can read it into R. First, load the following packages which we'll be using:
```{r, message=FALSE}
library(adehabitatHR)
library(tidyverse)
library(ggmap)
library(measurements)
library(stringr)
library(sf)
library(lubridate)
library(rgeos)
library(cowplot)
```

Let's read in the file, naming it `all_waypoints`
```{r}
all_waypoints <- read.csv("merged_waypoints.unicsv", header = F,
             col.names=paste0("V",seq_len(10)),fill=TRUE,na.strings=c(""))
```
Note the following parameters:

* `header = f` tells R that we don't want to read the first column as headers
* ` col.names=paste0("V",seq_len(10))` tells R that the data has 10 columns, and we're naming them V1-V10 for now
* `fill = TRUE` adds blank cells where rows are unequal length (our data has lots of uneven rows)
* `na.strings=c("")` converts blank spaces into NA values

\
Let's look at the data:
```{r}
head(all_waypoints)
```
The data are pretty messy. Use `View(all_waypoints)` if you want to look at the whole dataframe.

For cleaning the data, I mostly use the tidyverse package; here I convert the dataframe to what's called a "tibble", which is basically just a dataframe.
```{r}
all_waypoints <- as_tibble(all_waypoints)
```
\
When we converted the gdb files, we stored the file names as the last column of data. Due to differences in the gdb files, some files were converted with different number of columns. Let's make the last column, column 10, our new file name column with the below function:
```{r}
all_waypoints[,10] <- apply(all_waypoints, 1, function(x) tail(na.omit(x), 1))
```
\
Let's clean up the data, selecting the first 4 and the 10th column, which should contain the waypoint number, latitude, longitude, waypoint ID and the filename. Inspect the data again.
```{r}
all_waypoints_cleaned <- all_waypoints %>% select(1:4,10)
colnames(all_waypoints_cleaned) <- c("Number","Latitude","Longitude","Waypoint_ID","Filename")
head(all_waypoints_cleaned)
```
\
We've gotten rid of columns we don't need, but the headers of each individual file are still in the data. Note under the column names the tibble tells us the structure of each column. Currently latitude and longitude are "character" but should be "numeric". If we convert these to numeric, anything that is not a number (e.g., column headers from the old data we don't want) will be replaced by an NA.
```{r, warning = FALSE}
all_waypoints_cleaned <- all_waypoints_cleaned %>% mutate(Latitude = as.numeric(Latitude)) %>%
  mutate(Longitude = as.numeric(Longitude))
head(all_waypoints_cleaned)
```
Let's remove any rows that contain an NA, using `na.omit()` - after this the data looks a lot better.
```{r}
all_waypoints_cleaned <- na.omit(all_waypoints_cleaned)
```
\
\
Currently the file names don't correspond to the original file names - let's remove the end of the names
```{r}
all_waypoints_cleaned <- all_waypoints_cleaned %>%
  mutate(Filename = str_replace(Filename,"_converted.unicsv",""))
```

# Cleaning waypoint IDs

Let's store the number of waypoints we have currently in `original_waypoint_number`. Remove any unwanted characters from the waypoint ID column
```{r}
original_waypoint_number <- nrow(all_waypoints_cleaned)

# Removes spaces, brackets, colons, question marks, full stops, commas, apostrophes
all_waypoints_cleaned$Waypoint_ID <- gsub("[ ()-/:?.,']","", all_waypoints_cleaned$Waypoint_ID)
```
\
Below, we're trying to extract correctly formatted waypoint IDs into a new column: strings with four numbers (`\\d`) followed by SB, F, M, I, L, SM or IGI. We then make another column extracting only 4 numbers, as many waypoints don't have a letter after them.

```{r}
all_waypoints_cleaned$cleanedstring <- str_extract(all_waypoints_cleaned$Waypoint_ID, regex("\\d\\d\\d\\d(SB|F|M|I|L|SM|IGI)",ignore_case = TRUE))
all_waypoints_cleaned$cleanedstring2 <- str_extract(all_waypoints_cleaned$Waypoint_ID, regex("\\d\\d\\d\\d",ignore_case = TRUE))
```
\
Then we merge the columns together: if there is no waypoint extracted from the first attempt, then get the waypoint from the second attempt. We then remove the excess column. Finally, we remove any rows where a waypoint ID hasn't been located.
```{r}
all_waypoints_cleaned <- all_waypoints_cleaned %>%
  mutate(cleanedstring = if_else(!is.na(cleanedstring),cleanedstring,cleanedstring2)) %>%
  mutate(Time = str_extract(cleanedstring, regex("\\d\\d\\d\\d",ignore_case = TRUE))) %>% 
  select(!cleanedstring2)
all_waypoints_cleaned <- all_waypoints_cleaned %>% filter(!is.na(cleanedstring)) 
```

