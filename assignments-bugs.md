# Εργασίες σπιτιού (IDEA-06) — Bug Report

> Scope: το feature **εργασίες σπιτιού** (`Frontistirio-assignments-brain-repo`), δηλαδή `controllers/assignments.js`,
> `controllers/handlers/assignments.js`, `services/assignmentFiles.js` και οι σελίδες `/assignments*`, `/my-assignments`,
> `/my-children-assignments`.
>
> Βρέθηκαν **2026-09-21**, διαβάζοντας τον κώδικα (API `93b4079`, frontend `efc1260`) κατά τη συγγραφή του brain repo.
> **Δεν αναπαράχθηκαν σε ζωντανό περιβάλλον.** Το feature είναι live από 2026-09-21 (backend `93b4079`, frontend 1.5.17), οπότε το ASG-01 υπάρχει και σε production.

| ID | Severity | Status | Τίτλος |
|---|---|---|---|
| ASG-01 | MEDIUM | OPEN | Μια νέα παράδοση σβήνει τη διόρθωση, και ο server δεν το εμποδίζει |
| ASG-02 | LOW | OPEN (δεν επαληθεύτηκε) | Ελληνικά ονόματα αρχείων στο `Content-Disposition` χωρίς κωδικοποίηση RFC 5987 |
| ASG-03 | LOW | OPEN | Η λίστα μαθητών υπολογίζεται ζωντανά: ένας μαθητής που αλλάζει τμήμα χάνει την εργασία και η παράδοσή του κρύβεται |
| ASG-04 | LOW | OPEN | Το `edit-assignment` αντιμετωπίζει τα πεδία που λείπουν ως τιμές |

---

## ASG-01 · MEDIUM — Μια νέα παράδοση σβήνει τη διόρθωση, και ο server δεν το εμποδίζει

**Αρχεία:** `controllers/assignments.js` → `submitAssignment`, `controllers/handlers/assignments.js` → `upsertSubmission`.

Το `upsertSubmission`, όταν υπάρχει ήδη παράδοση, κάνει `status = "submitted"`, `score = null`, `feedback = ""`,
`gradedAt = null`, `gradedBy = undefined` (σκόπιμα, γιατί «ο καθηγητής διόρθωσε κάτι που δεν υπάρχει πια»).
Όμως το `submitAssignment` **δεν ελέγχει** αν η υπάρχουσα παράδοση είναι `graded`. Αντίθετα, το `withdrawSubmission`
την αρνείται (`submission_already_graded`).

Το UI κρύβει τον editor όταν η κατάσταση είναι `graded` (`my-assignments.component.html`, `*ngIf="row.state !== 'graded'"`).
Άρα η μόνη προστασία είναι στο client. Αν ένας μαθητής στείλει POST στο `/assignments/submit-assignment` για εργασία που έχει
διορθωθεί (και δεν έχει κλειδώσει με `allowLate:false`), **η διόρθωση του καθηγητή χάνεται** χωρίς ίχνος.

**Fix:** στο `submitAssignment`, μετά το `findSubmission`:
`if (existing?.status === "graded") throw new Error("submission_already_graded");`
Το ίδιο μήνυμα υπάρχει ήδη στο i18n του client. Αν αργότερα επιτραπεί ρητά η «επανυποβολή μετά από διόρθωση», να γίνει με
σημαία στην εργασία και όχι σιωπηλά.

---

## ASG-02 · LOW (δεν επαληθεύτηκε) — Ελληνικά ονόματα αρχείων στο `Content-Disposition`

**Αρχείο:** `services/assignmentFiles.js` → `signedDownloadUrl`.

`ResponseContentDisposition: attachment; filename="${ref.filename}"`. Το όνομα είναι UTF-8 (έχει ήδη μετατραπεί από latin1 στο upload)
και μπαίνει ωμό, χωρίς `filename*=UTF-8''<percent-encoded>`. Το S3 το επιστρέφει όπως είναι. Αν ένας browser ή WebView το διαβάσει ως
latin1, το αρχείο αποθηκεύεται με χαλασμένο όνομα. Ένα όνομα με `"` σπάει το header.

**Fix:** `attachment; filename="<ascii fallback>"; filename*=UTF-8''${encodeURIComponent(name)}`. Ο ίδιος κώδικας πιθανότατα υπάρχει και στις
λήψεις του εκπαιδευτικού υλικού: να ελεγχθεί εκεί.

---

## ASG-03 · LOW — Η λίστα μαθητών υπολογίζεται ζωντανά

**Αρχεία:** `handlers/assignments.js` → `buildAssignmentRoster`, `getStudentAssignments`, `isStudentTargeted`.

Ο στόχος «τμήμα» λύνεται **τη στιγμή του αιτήματος** μέσω του `Student.period_class`. Αν ένας μαθητής μετακινηθεί σε άλλο τμήμα μέσα
στην περίοδο:
- δεν βλέπει πια την εργασία στο `/my-assignments`, ούτε μπορεί να κατεβάσει το συνημμένο της
- χάνεται από την οθόνη διόρθωσης, οπότε **η παράδοσή του (και ο βαθμός) δεν φαίνεται πουθενά**, παρότι υπάρχει στη βάση
- αντίθετα, ο μαθητής «κληρονομεί» τις εργασίες του νέου τμήματος, ακόμα και όσες ανατέθηκαν πριν μπει σε αυτό

**Fix (επιλογή προϊόντος):** είτε (α) η οθόνη διόρθωσης να δείχνει επιπλέον όσους **έχουν παράδοση** αλλά δεν είναι πια στη λίστα
(ένωση roster ∪ υποβολές), είτε (β) να κρατιέται στιγμιότυπο της λίστας μαθητών κατά τη δημοσίευση. Το (α) είναι φθηνό και δεν χάνει δεδομένα.

---

## ASG-04 · LOW — Το `edit-assignment` αντιμετωπίζει τα πεδία που λείπουν ως τιμές

**Αρχείο:** `controllers/assignments.js` → `editAssignment`.

`dueAt: body.dueAt ? new Date(body.dueAt) : null`, `allowLate: parseBool(body.allowLate, true)`, `requireFile: parseBool(…, false)`,
`visibleToParents: parseBool(…, true)`, `status: body.status === "draft" ? "draft" : "published"`, `maxScore: parseNumberOrNull(…)`.
Ένα αίτημα που στέλνει μόνο τα πεδία που άλλαξαν **σβήνει την προθεσμία**, **δημοσιεύει ένα draft** και **κάνει τον βαθμό «μόνο σχόλιο»**.

Ο μοναδικός client (`add-assignment.component.ts#save`) στέλνει πάντα ολόκληρο το payload, οπότε σήμερα δεν χτυπάει. Είναι όμως παγίδα
για κάθε μελλοντικό client (π.χ. ένα κουμπί «δημοσίευση» στη λίστα).

**Fix:** για κάθε πεδίο, `if (body.x !== undefined)`, όπως κάνει ήδη το `keptAttachments` (`null` = κράτα ό,τι υπάρχει).
