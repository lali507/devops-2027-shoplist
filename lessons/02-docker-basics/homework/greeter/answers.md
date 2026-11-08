# שיעור 2: תשובות על הפקודות

**1. בניית ה-image מהתיקייה הראשית של הריפו:**

```shell
docker build -t greeter:v1 lessons/02-docker-basics/homework/greeter
```

**2. הרצה ברקע, בשם `greeter`, דרך `http://localhost:9000`:**

```shell
docker run -d -p 9000:8000 --name greeter greeter:v1
```

**3. ברכה `Shalom` במקום `Hello`:** מוסיפים `-e GREETING=Shalom` לפני שם ה-image:

```shell
docker run -d -p 9000:8000 --name greeter -e GREETING=Shalom greeter:v1
```

**4. מה תחזיר `http://localhost:9000/hello/Rivka`:**

```json
{"message":"Shalom, Rivka!"}
```

**5. שינוי שורה ב-`app.py` ובנייה מחדש:** השלבים `FROM`, `WORKDIR`, `COPY requirements.txt` ו-`RUN pip install` יסומנו `CACHED`, כי הם והקבצים שלהם לא השתנו. השלב `COPY app.py` ירוץ מחדש כי תוכן הקובץ השתנה, וכל מה שאחריו נבנה מחדש, כי ברגע שה-cache נשבר בשלב אחד הוא לא תקף גם לשלבים שאחריו.
