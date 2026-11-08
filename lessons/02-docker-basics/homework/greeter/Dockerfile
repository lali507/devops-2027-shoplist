# מתחילים מ-image של Python בגרסה מדויקת, כדי שכל build ייתן אותה סביבה
FROM python:3.11-slim

# קובעים את /app כתיקיית העבודה של כל הפקודות הבאות
WORKDIR /app

# מעתיקים קודם רק את רשימת התלויות, כדי שה-cache של ההתקנה יישמר כשהקוד משתנה
COPY requirements.txt .

# מתקינים את התלויות, והדגלים מאפשרים ל-pip לעבוד גם ברשת של נטפרי
RUN pip install --no-cache-dir --trusted-host pypi.org --trusted-host files.pythonhosted.org -r requirements.txt

# מעתיקים את הקוד אחרי ההתקנה, כי הוא הקובץ שמשתנה הכי הרבה
COPY app.py .

# מתעדים שהשרת מאזין בפורט 8000 בתוך ה-container
EXPOSE 8000

# מריצים את השרת בצורת exec כשה-container עולה
CMD ["python", "app.py"]
