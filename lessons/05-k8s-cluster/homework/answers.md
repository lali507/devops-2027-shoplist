<div dir="rtl">

# שיעור 5 — תשובות

## חלק א — מפת ה-cluster

| רכיב | איפה הוא רץ (הנהלה / כל node) | מה הוא עושה, במשפט אחד | אנלוגיה |
|------|-------------------------------|------------------------|---------|
| API server | הנהלה (control plane) | הכניסה היחידה לקלאסטר: מקבל בקשות מ-kubectl ומכל הרכיבים, בודק אותן, והוא היחיד שקורא וכותב ל-etcd | פקיד הקבלה / טלפון המשרד |
| etcd | הנהלה (control plane) | מסד הנתונים (key-value) של הקלאסטר: שומר את כל המצב, גם מה שצריך להיות וגם מה שקיים בפועל. רק ה-API server ניגש אליו. זה לא ה-DB של האפליקציה | טבלת הרישום במשרד |
| scheduler | הנהלה (control plane) | מחליט לכל pod חדש על איזה node הוא ירוץ, לפי המשאבים הפנויים בכל node. הוא רק מחליט ולא מריץ בעצמו | מזכירה שמחליטה איזו משימה הולכת לאיזה חדר |
| controller manager | הנהלה (control plane) | בודק כל הזמן אם מה שקיים בפועל תואם למה שצריך להיות, ומתקן כשיש פער (למשל, יוצר pod חדש כשאחד נפל) | מנהלת שעוברת כל כמה שניות על המשרד, ואם חסרה עובדת היא מגייסת מחליפה |
| kubelet | כל node | מקבל מה-API server אילו pods צריכים לרוץ על ה-node שלו, מבקש מה-container runtime להריץ אותם, ומדווח שה-node וה-pods חיים | אחראית חדר במשרד שמחלקת את העבודה בחדר ומדווחת שהחדר פעיל |
| container runtime | כל node | מוריד את ה-images ומריץ בפועל את הקונטיינרים של ה-pods, לפי מה שה-kubelet מבקש (למשל containerd) | העובדות בחדרים שעושות את העבודה בפועל |
| kube-proxy | כל node | מנתב תעבורה שמגיעה לכתובת של Service אל אחד ה-pods שמאחוריו, ובכך גם מחלק עומס ביניהם | מרכזייה שמעבירה שיחה למספר של מחלקה אל אחת העובדות הפנויות בה |
| coredns | לא על כל node: רץ כ-pods רגילים ב-namespace בשם kube-system (בדרך כלל 2 עותקים) | ה-DNS של הקלאסטר: מתרגם שם של Service (למשל `postgres`) לכתובת ה-IP שלו, כדי שה-pods יוכלו לפנות זה לזה לפי שם | ספר הטלפונים של המשרד: מחפשים לפי שם ומקבלים מספר |

---

## חלק ב — תרחישים

### 1. scale
רצים 2 pods של Deployment בשם `web`, ומריצות `kubectl scale deployment web --replicas=4`. תארו לפי הסדר מה קורה, מהרגע שלחצתן Enter ועד שיש 4 pods ב-Running. לכל שלב: איזה רכיב פועל?

**תשובה:**
1. **API server:** מקבל את הבקשה מ-kubectl, בודק אותה, וכותב ל-etcd שצריכים להיות 4 pods של `web`.
2. **controller manager** (בהנהלה): משווה בין מה שצריך (4) למה שקיים (2), ויוצר דרך ה-API server שני pods חדשים. בשלב הזה עוד אין להם node.
3. **scheduler:** רואה שני pods בלי node, ומחליט לכל אחד על איזה node הוא ירוץ.
4. **kubelet** בכל node שנבחר: רואה שהוקצה לו pod חדש, ומבקש מה-**container runtime** להוריד את ה-image ולהריץ את הקונטיינר.
5. **kubelet:** מדווח ל-API server שה-pods במצב Running. עכשיו רצים 4: שני הישנים שלא נגעו בהם, ושניים חדשים. אם יש Service ל-`web`, ה-**kube-proxy** מתחיל להעביר תעבורה גם לשני החדשים.

### 2. שלוש בלילה
ה-container של backend קרס בשלוש בלילה. מה קורה? ומה היה קורה אם מישהי הייתה מוחקת את ה-pod כולו?

**תשובה:**

**כשרק ה-container קרס:**
1. **kubelet** ב-node שבו רץ ה-pod שם לב שהקונטיינר יצא (נפל).
2. לפי ברירת המחדל (`restartPolicy: Always`) ה-kubelet מבקש מה-**container runtime** להרים את אותו קונטיינר מחדש, באותו pod ובאותו node. ה-pod לא נמחק ולא עובר node.
3. ה-kubelet מדווח ל-API server, ושם עולה המונה `RESTARTS` ב-`kubectl get pods`.
4. אם הקונטיינר ממשיך ליפול, ה-kubelet מאריך את ההשהיה בין ניסיון לניסיון, וה-pod נראה במצב `CrashLoopBackOff`.

**אם מישהי מוחקת את ה-pod כולו** (ה-pod שייך ל-Deployment):
1. **API server** מוחק את ה-pod מ-etcd, וה-kubelet עוצר את הקונטיינרים שלו.
2. **controller manager** (ה-ReplicaSet שמתחת ל-Deployment) רואה שיש פחות pods ממה שצריך, ויוצר pod חדש.
3. **scheduler** בוחר לו node (לא בהכרח אותו node).
4. **kubelet** באותו node מריץ את הקונטיינר דרך ה-container runtime.

ההבדל: בקריסה של קונטיינר, זה אותו pod שמופעל מחדש. במחיקת pod, נוצר pod חדש לגמרי, עם שם חדש וכתובת IP חדשה.

### 3. node נכבה
ב-cluster בענן יש 3 nodes, ו-node 2 נכבה. מה קורה ל-pods שרצו עליו? ומה היה קורה ב-cluster של Docker Desktop בכיתה, אם המחשב נכבה?

**תשובה:**

**ב-cluster בענן (3 nodes):**
1. ה-kubelet של node 2 הפסיק לדווח ל-API server שהוא חי.
2. **controller manager** (ה-node controller שבו) מזהה שאין דיווחים, ואחרי כמה עשרות שניות מסמן את ה-node כ-`NotReady`.
3. ה-pods שרצו עליו עדיין מופיעים, אבל לא נותנים שירות. אחרי זמן המתנה (ברירת מחדל: כ-5 דקות) הם מסומנים למחיקה.
4. ה-ReplicaSet רואה שחסרים pods, ויוצר pods חדשים. ה-**scheduler** מציב אותם על node 1 ו-node 3, וה-kubelet שם מריץ אותם.
5. ה-**kube-proxy** מפסיק להעביר תעבורה ל-pods שלא בריאים, והמשתמשים מגיעים רק ל-pods שעל ה-nodes שחיים.

התוצאה: האפליקציה ממשיכה לעבוד על שני ה-nodes שנשארו (אולי עם הפסקה קצרה, או בעומס גבוה יותר). זו הסיבה שמריצים כמה replicas על כמה nodes. שימי לב שזה נכון רק ל-pods שנוצרו דרך Deployment. pod בודד שרץ על ה-node הזה פשוט אבד.

**ב-Docker Desktop בכיתה:**
יש node אחד בלבד, והוא גם ההנהלה וגם המקום שבו ה-pods רצים. אם המחשב נכבה, הכל נופל יחד: ה-API server, ה-etcd וה-pods. אין node אחר שיכול להרים אותם, ולכן האפליקציה לא זמינה כל עוד המחשב כבוי. כשהמחשב עולה וה-Docker Desktop מתחיל, ה-cluster עולה, המצב הרצוי נשמר ב-etcd, וה-Deployments מרימים את ה-pods מחדש (כקונטיינרים חדשים, כך שמידע שלא נשמר ב-volume ילך לאיבוד).

### 4. מהכיתה לענן
חברה רוצה להעביר את ShopList ל-EKS, עם אותם קבצי YAML שעבדו בכיתה: `imagePullPolicy: Never` ו-Service מסוג `NodePort`. מה לא יעבוד, ולמה? מה צריך לשנות?

**תשובה:**

**1. `imagePullPolicy: Never`**
- **למה זה עבד בכיתה:** ה-image נבנה מקומית ב-Docker Desktop, והוא כבר היה על ה-node. ה-`Never` אומר ל-kubelet לא להוריד שום דבר ולהשתמש רק במה שקיים על ה-node.
- **למה זה לא יעבוד ב-EKS:** ה-nodes הם מכונות נפרדות בענן, ואין עליהן את ה-image. ה-pod ייתקע במצב `ErrImageNeverPull`.
- **מה לשנות:**
  - לדחוף (push) את ה-image ל-registry, למשל ECR של AWS.
  - לכתוב ב-YAML את השם המלא של ה-image כולל ה-registry והתג.
  - להסיר את `Never` ולהשתמש ב-`IfNotPresent` (או `Always`).
  - לוודא שלמכונות ה-nodes יש הרשאה למשוך מה-registry (ב-ECR זה נעשה דרך הרשאות IAM של ה-node group, ואם ה-registry פרטי אחר, דרך `imagePullSecrets`).

**2. Service מסוג `NodePort`**
- **למה זה עבד בכיתה:** ה-node הוא המחשב שלנו, ואפשר לגשת אליו דרך `localhost:<nodePort>`.
- **למה זה לא מתאים ל-EKS:** ה-nodes נמצאים ברשת פרטית בענן, כתובות ה-IP שלהם משתנות כשהם מוחלפים, ו-security group חוסם בדרך כלל את הפורט. משתמשים אמיתיים לא אמורים לפנות לכתובת של node ספציפי.
- **מה לשנות:** להשתמש ב-Service מסוג `LoadBalancer` (EKS יוצר Load Balancer של AWS עם כתובת קבועה שמחלקת עומס בין ה-nodes), או ב-Ingress מול Service רגיל (`ClusterIP`) כשיש כמה שירותים ורוצים כתובת אחת ונתיבים.

(אם ל-ShopList יש מסד נתונים עם אחסון, צריך גם לבדוק את ה-volumes: ב-EKS צריך `StorageClass` של הענן, למשל EBS, במקום האחסון המקומי של הכיתה.)

### 5. pod בודד
בכיתה יצרתן pod עם `kubectl run`, מחקתן אותו, והוא לא חזר. אבל כשמחקתן pod של `web`, הוא חזר. למה? מה הייתן עושות כדי ש-stats יחזור תמיד?

**תשובה:**

`kubectl run` יוצר pod בודד, בלי שום בקר (controller) שאחראי עליו. כשמוחקים אותו, אין מי שיבדוק שהוא חסר, כי אף אחד לא הגדיר "צריך שיהיה 1 כזה".

ה-pods של `web` נוצרו דרך Deployment. ה-Deployment יוצר ReplicaSet, וה-**controller manager** בודק כל הזמן שמספר ה-pods שקיימים שווה ל-`replicas`. כשמחקנו pod, הוא ראה פער ויצר pod חדש (ואז ה-scheduler וה-kubelet הריצו אותו).

**כדי ש-stats יחזור תמיד:** לא להריץ אותו כ-pod בודד, אלא לעטוף אותו ב-Deployment עם `replicas: 1` לפחות:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: stats
spec:
  replicas: 1
  selector:
    matchLabels:
      app: stats
  template:
    metadata:
      labels:
        app: stats
    spec:
      containers:
        - name: stats
          image: stats:latest   # בענן: שם מלא מה-registry
```

ואז `kubectl apply -f stats.yaml`. עכשיו גם אם ה-pod יימחק או ה-node שלו ייפול, ה-controller manager יוצר אותו מחדש.

---

## בונוס
מה יקרה אם תשני ב-Deployment את `replicas` ל-0 ותריצי `kubectl apply`? האם ה-Service ימשיך להתקיים? מה יקבל הדפדפן?

**תשובה:**

- ה-**API server** שומר ב-etcd את המצב החדש (0 עותקים). ה-**controller manager** רואה שיש pods שלא אמורים להתקיים, ומוחק את כולם. ה-kubelet עוצר את הקונטיינרים.
- ה-**Deployment** עצמו נשאר קיים, רק בלי pods. אפשר להחזיר אותו לחיים בשינוי חזרה של `replicas`.
- ה-**Service** ממשיך להתקיים: הוא אובייקט נפרד שלא תלוי בכמה pods יש, והוא שומר על אותה כתובת ועל אותו פורט. אבל אין לו pods שמאחוריו (רשימת ה-endpoints ריקה), ולכן ה-kube-proxy אין לו לאן להעביר את התעבורה.
- **הדפדפן** לא יקבל דף מהאפליקציה, ובוודאי לא שגיאה של האפליקציה. הוא יקבל כשל חיבור: "לא ניתן להתחבר" (`ERR_CONNECTION_REFUSED` או חיבור שנתקע ונגמר הזמן). אם יש Load Balancer או Ingress מול ה-Service, אפשר לראות שם שגיאה כמו 503.

</div>
