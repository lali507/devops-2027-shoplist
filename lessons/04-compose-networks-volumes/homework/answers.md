# שיעור 4: תשובות (ד10)

**1.** stats מגיע ל-backend בתוך הרשת של Compose לפי השם, בכתובת `http://backend:5000`, ולכן לא צריך `ports`. הדפדפן לא מגיע אליו ישירות: הוא פונה ל-frontend בפורט 8080, ו-Nginx מעביר את הבקשות של `/api/` אל backend.

**2.** אחרי `down` ו-`up` המוצרים נשארים, כי הם שמורים ב-named volume בשם `postgres-data`, ו-`down` מוחק רק את ה-containers ואת הרשת. `down -v` מוחק גם את ה-volume, ולכן המוצרים נמחקים.

**3.** זה bind mount, ויודעים את זה כי הצד השמאלי הוא נתיב במחשב ולא שם של volume. `init.sql` רץ רק כשתיקיית הנתונים ריקה, ולכן שינוי אחרי שהמסד כבר נוצר לא ישפיע עד `down -v` ו-`up`.

**4.** הרשת נקראת `docker_default` וה-volume נקרא `docker_postgres-data`. Compose מוסיף לשמות את שם הפרויקט, שהוא `docker` כי זה שם התיקייה של קובץ ה-Compose. `default` הוא שם הרשת האוטומטית, ו-`postgres-data` הוא השם שנתתי ל-volume.

**5.** ב-image הסופי יש Nginx, את הקבצים הבנויים מ-`dist` ואת `nginx.conf`. אין בו Node, `build.js` או קובצי המקור. אין `CMD` כי ה-`CMD` של Nginx עובר בירושה מה-image הבסיסי. ל-stats לא צריך multi-stage כי אין לו שלב בנייה, ו-Node נדרש לו גם בזמן הריצה.

**6.** הגדלים: `single` הוא 62.4MB, `multi` הוא 62.4MB ו-`kitchen` הוא 172MB. `multi` כמעט בגודל של `single` כי שניהם מבוססים על אותו Nginx עם אותם קבצים קטנים, ו-Node נשאר בשלב הבנייה ולא נכנס ל-image הסופי.

**7.** Nginx בוחר את ה-prefix הארוך ביותר שמתאים, ו-`/api/stats` ארוך יותר מ-`/api/`, ולכן הבקשה מגיעה ל-stats. בגלל הלוכסן בסוף `http://backend:5000/`, החלק `/api/` מוחלף ב-`/`, כך שהבקשה `/api/products` מגיעה ל-backend בתור `/products`.
