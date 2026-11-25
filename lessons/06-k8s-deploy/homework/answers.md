<div dir="rtl">

# שיעור 6 — תשובות

## צילומי מסך (ד9)

| קובץ | מה רואים |
|------|----------|
| [screenshots/app.png](screenshots/app.png) | ShopList ב-`localhost:30080`, שלושה מוצרים, שתי נוריות ירוקות, ושורת `Build:` בתחתית (ד11) |
| [screenshots/get-all.png](screenshots/get-all.png) | `kubectl get all -n shoplist`: חמישה pods ב-`1/1 Running` (שניים של backend, בגלל `replicas: 2`) |
| [screenshots/survived.png](screenshots/survived.png) | אחרי `kubectl delete pod -l app=postgres`: ה-pod של postgres חדש (AGE של 16s), והמוצרים עדיין בדפדפן |

---

## ד10 — שאלות

### 1. למה `shoplist-backend:local` ולא `docker-backend`? ומה בלי `imagePullPolicy: Never`?
את השם `docker-backend` Compose ממציא בעצמו, מהשם של התיקייה ומהשם של השירות. אם נשנה תיקייה, השם ישתנה, ולכן הוא לא שם שאפשר לסמוך עליו מתוך קובץ YAML. עם `-t` אנחנו קובעות שם ותג קבועים.
בלי `imagePullPolicy: Never` (וכשהתג הוא לא `latest`, ברירת המחדל היא `IfNotPresent`), Kubernetes היה מחפש את ה-image ב-Docker Hub אם הוא לא מוצא אותו מקומית. ה-image לא קיים שם, אז היינו מקבלות `ErrImagePull` ואחר כך `ImagePullBackOff`. עם `Never` הוא משתמש רק במה שיש במחשב, ואם חסר image מקבלים `ErrImageNeverPull`. זה בדיוק מה שקרה לי בהתחלה, לפני שבניתי את ה-images בבית.

### 2. פענוח `POSTGRES_PASSWORD`
```shell
kubectl get secret postgres-secret -n shoplist -o jsonpath='{.data.POSTGRES_PASSWORD}' | base64 -d
```
התוצאה היא `shoppass`. ‏base64 הוא קידוד ולא הצפנה: כל מי שיכול לקרוא את ה-Secret מה-cluster או מ-Git יכול לפענח אותו בשנייה. ההגנה האמיתית מגיעה מהרשאות (RBAC), מהצפנה של etcd, ומזה שלא עושים commit לסיסמאות אמיתיות.

### 3. המוצרים שרדו. מי אחראי, ומה כן היה מוחק אותם?
`postgres-pvc.yml`: ה-PVC מחזיק את הדיסק, והוא חי בנפרד מה-pod. ב-`postgres-deployment.yml` הדיסק מחובר ל-`/var/lib/postgresql/data`. ה-pod החדש התחבר לאותו דיסק.
מה כן היה מוחק אותם: `kubectl delete pvc postgres-pvc -n shoplist`, או `kubectl delete namespace shoplist`, שמוחק את כל מה שבתוכו וגם את ה-PVC.

### 4. אין `depends_on`. מה קורה אם backend עולה לפני postgres?
ה-backend שלנו לא נופל: הוא מתחבר ל-DB רק כשמגיעה בקשה (`get_db()`), ולא כשהוא עולה. אם postgres עוד לא מוכן, בקשה ל-`/products` תיכשל עם שגיאה, אבל הבקשה הבאה, אחרי ש-postgres עלה, כבר תעבוד. כלומר, המצב מתקן את עצמו.
מה שדואג שה-backend לא יקבל בקשות לפני שהוא עצמו מוכן הוא ה-**readinessProbe** (`/health`). עד שהוא עונה 200, ה-pod לא נכנס ל-endpoints של ה-Service, ו-Nginx לא יכול להגיע אליו. ב-postgres יש readinessProbe עם `pg_isready`, אז גם ה-Service של postgres לא מפנה לשם תעבורה עד שה-DB באמת מקבל חיבורים. ואם קונטיינר כן נופל, Kubernetes מרים אותו שוב ושוב עד שהכל מסתדר. במקום סדר עלייה יש "לנסות שוב עד שזה עובד".
(הערה: ה-`/health` שלנו לא בודק את ה-DB, הוא רק עונה `ok`. אז readiness כאן אומר "Flask חי", לא "יש DB".)

### 5. ד8: selector שגוי (`app: backnd`)
- **בדפדפן:** הנורית של backend הפכה לאדומה, ורשימת המוצרים לא נטענה. ‏`/api/products` ו-`/api/health` החזירו `502 Bad Gateway` מ-Nginx. הנורית של stats נשארה ירוקה, כי stats עצמו חי וענה `{"error":"backend unavailable"}`, אבל הסטטיסטיקות לא נטענו.
- **ה-pods של backend:** לא נפלו. שניהם נשארו `1/1 Running` עם 0 restarts.
- **`kubectl get endpoints backend`:** הראה `<none>`. ב-`describe service` ראינו `Selector: app=backnd` ושורת `Endpoints` ריקה.
- **למה:** ה-Service מוצא את ה-pods שלו רק לפי המדבקות (labels). אין אף pod עם `app=backnd`, אז ל-Service אין לאן לשלוח בקשות, גם כשה-pods בריאים לגמרי. אחרי התיקון ל-`app: backend` שתי הכתובות חזרו ל-endpoints, והנוריות חזרו לירוק.

### 6. שיניתי את `app-config`, אבל backend רואה את הערך הישן
משתני סביבה נקראים רק פעם אחת, כשהקונטיינר עולה. `apply` מעדכן את ה-ConfigMap ב-cluster, אבל לא נוגע ב-pods שכבר רצים, וה-Deployment עצמו לא השתנה, אז הוא לא מחליף pods. הפתרון הוא `kubectl rollout restart deployment/backend -n shoplist`, שמרים pods חדשים שקוראים את הערך החדש.

### 7. כמה פעמים כתוב `shoplist`?
ב-12 הקבצים המקוריים (בלי `app-config.yml`) המילה `shoplist` מופיעה 19 פעמים. 12 מהן הן שם ה-namespace: ‏`namespace: shoplist` ב-11 קבצים, ו-`name: shoplist` ב-`namespace.yml`. השאר הם שמות images, שם ה-DB ב-`pg_isready`, והערות.
כדי להריץ עותק ב-`shoplist-test` צריך לשנות את **כל 12 הקבצים** (13 עם `app-config.yml`). אפשר גם להוריד את `namespace:` מהקבצים ולהריץ `kubectl apply -n shoplist-test`, אבל עדיין כל ערך שחוזר על עצמו כתוב בהרבה מקומות. זה בדיוק מה ש-Helm פותר: כותבים את הערך פעם אחת, ב-`values.yaml`, ומשתמשים בו בכל התבניות.

---

## בונוס

### ד11 — frontend מ-multi-stage
בניתי את `shoplist-frontend:multi` מ-`Dockerfile.multistage`, ושיניתי את ה-image ב-`frontend-deployment.yml`. ה-pod של frontend הוחלף בחדש, ובתחתית הדף מופיעה השורה `Build: 2026-11-25T16:33:58.500Z`.
**מה קרה ל-pod הישן:** ה-Deployment עשה rolling update. הוא יצר ReplicaSet חדש עם pod חדש, וחיכה שהחדש יהיה `Running`/Ready. רק אז הוא הוריד את הישן ל-0 ומחק אותו (רואים את זה ב-`get-all.png`: ‏`frontend-8c7586546` עם 0, ו-`frontend-6676575f64` עם 1). לכן כמעט לא היה רגע שבו האפליקציה לא עבדה. ל-frontend אין readinessProbe, אז Kubernetes מחשיב אותו מוכן ברגע שהקונטיינר עלה, ובתיאוריה יכול להיות חלון קצרצר עד ש-Nginx באמת מקשיב.

### ד12 — livenessProbe ל-backend
הוספתי `livenessProbe` מסוג `httpGet` ל-`/health:5000`, עם `initialDelaySeconds: 15`, `periodSeconds: 20` ו-`failureThreshold: 3`. ב-`kubectl describe pod -l app=backend` מופיעות עכשיו שתי השורות, `Liveness:` ו-`Readiness:`.
- **רק readiness:** backend תקוע מפסיק לענות, ה-readiness נכשל, וה-pod יוצא מה-endpoints ולא מקבל יותר תעבורה. אבל הוא נשאר תקוע לנצח. אף אחד לא מאתחל אותו, ואם הוא היחיד, האפליקציה למטה.
- **גם liveness:** אחרי 3 כישלונות (כ-60 שניות) ה-kubelet הורג את הקונטיינר ומרים אותו מחדש. כלומר, ריפוי עצמי.
- **למה liveness לא יבדוק את postgres:** אם postgres נופל, כל ה-pods של backend ייכשלו ב-liveness ויאותחלו שוב ושוב (CrashLoop), אבל אתחול של backend לא מתקן את postgres. זה רק מוסיף עומס ורעש. liveness צריך לשאול רק "התהליך שלי חי?". "התלויות שלי זמינות?" היא שאלה של readiness.

</div>
