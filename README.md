# lectuer17-2569-starter — React Hook Form + Zod

---

## คู่มือ : https://ui.shadcn.com/docs/forms/react-hook-form

## ขั้นตอนที่ 12: ติดตั้ง

### 12.1 ติดตั้ง dependency เดิมของโปรเจกต์

```bash
pnpm install
pnpm run dev
```

## โครงสร้างไฟล์ (เฉพาะที่เกี่ยวกับ Lab นี้)

| ไฟล์                                                 | หน้าที่                                                                                                |
| ---------------------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| `src/lib/types.ts`                                   | `Student` (+ `interests?`, `emails?`), `StudentEmail`, `Course`, `Enrollment`                          |
| `src/lib/mock-data.ts`                               | ข้อมูลตั้งต้น (นักศึกษามี interests / emails แล้ว)                                                     |
| `src/lib/enrollment-store.ts`                        | Zustand store — key `lecture17-starter-storage`                                                        |
| `src/lib/schemas/student-schema.ts`                  | `interestOptions`, `MAX_EMAILS`, `studentFormSchema`, `StudentFormValues`, `createStudentFormSchema()` |
| `src/components/students/add-new-student-dialog.tsx` | ปุ่ม + popup ฟอร์มเพิ่มนักศึกษา (React Hook Form + `useFieldArray`)                                    |
| `src/pages/admin/students.tsx`                       | หน้าจัดการนักศึกษา (ปุ่ม popup + ตาราง)                                                                |

### 12.2 ติดตั้ง React Hook Form + Zod

```bash
pnpm add zod react-hook-form @hookform/resolvers
```

| Package               | หน้าที่                                                                               |
| --------------------- | ------------------------------------------------------------------------------------- |
| `zod`                 | ประกาศ "กฎ" ของข้อมูลเป็น schema แล้วได้ TypeScript type มาจาก schema เลย             |
| `react-hook-form`     | จัดการ state ของฟอร์ม (ค่า, error, touched, submit) และ array field (`useFieldArray`) |
| `@hookform/resolvers` | ตัวเชื่อม — ให้ React Hook Form ใช้ Zod schema ตรวจค่า (`zodResolver`)                |

### 12.3 ติดตั้ง shadcn `field` component

```bash
pnpm dlx shadcn@latest add field
```

ระหว่างติดตั้งจะถาม 2 ครั้ง ให้ตอบ **`N`** ทั้งสองครั้ง

```text
? The file label.tsx already exists. Would you like to overwrite? » (y/N)      ← N
? The file separator.tsx already exists. Would you like to overwrite? » (y/N)  ← N
```

> 💡 `field.tsx` ให้ `<Field>`, `<FieldLabel>`, `<FieldError>`, `<FieldSet>`, `<FieldLegend>` ... ที่ใช้คู่กับ `<Controller>`

---

## ขั้นตอนที่ 13: เพิ่ม field ใหม่ใน Type และ Mock Data

### 13.1 แก้ `src/lib/types.ts`

```ts
interface Student {
  studentId: string;
  firstName: string;
  lastName: string;
  program: "CPE" | "ISNE";
  courses?: string[];
  // ความสนใจ (id จาก interestOptions) — optional เพราะข้อมูลที่ persist ไว้ก่อนหน้าอาจยังไม่มี
  interests?: string[];
  // อีเมลของนักศึกษา (หลายอีเมลได้)
  emails?: StudentEmail[];
}
export type { Student };

interface StudentEmail {
  address: string;
}
export type { StudentEmail };
```

### 13.2 แก้ `src/lib/mock-data.ts`

เพิ่ม `interests` และ `emails` ให้นักศึกษาทั้ง 3 คน:

```ts
export const students: Student[] = [
  {
    studentId: "650610001",
    firstName: "Matt",
    lastName: "Damon",
    program: "CPE",
    interests: ["web", "mobile"],
    emails: [{ address: "matt.damon@cmu.ac.th" }],
  },
  {
    studentId: "650610002",
    firstName: "Cillian",
    lastName: "Murphy",
    program: "CPE",
    courses: ["261207", "261497"],
    interests: ["ai"],
    emails: [
      { address: "cillian.murphy@cmu.ac.th" },
      { address: "cillian.m@gmail.com" },
    ],
  },
  {
    studentId: "650610003",
    firstName: "Emily",
    lastName: "Blunt",
    program: "ISNE",
    courses: ["269101", "261497"],
    interests: ["network", "web", "ai"],
    emails: [{ address: "emily.blunt@cmu.ac.th" }],
  },
];
```

---

## ขั้นตอนที่ 14: สร้าง Zod schema ของฟอร์มนักศึกษา

สร้างไฟล์ใหม่ `src/lib/schemas/student-schema.ts` — ใช้แทน `student-validation.ts` (if-else) ของ lecture16

### 14.1 ค่าคงที่ + ตัวเลือกความสนใจ

```ts
import { z } from "zod";

import type { Student } from "@/lib/types";

export const MAX_INTERESTS = 3;
export const MAX_EMAILS = 3;

// ใช้ร่วมกันระหว่างฟอร์ม (Checkbox) กับตารางจัดการนักศึกษา (แสดง label)
export const interestOptions = [
  { id: "web", label: "Web Development" },
  { id: "mobile", label: "Mobile Application" },
  { id: "ai", label: "AI / Machine Learning" },
  { id: "network", label: "Network & Security" },
];
```

### 14.2 Schema — กฎของทุกช่องในที่เดียว

```ts
export const studentFormSchema = z.object({
  studentId: z
    .string()
    .trim()
    .regex(/^\d{9}$/, "รหัสนักศึกษาต้องเป็นตัวเลข 9 หลัก"),
  firstName: z.string().trim().min(1, "กรอกชื่อ"),
  lastName: z.string().trim().min(1, "กรอกนามสกุล"),
  program: z.enum(["CPE", "ISNE"], { message: "เลือกหลักสูตร" }),

  // Checkbox หลายตัว → array ของ id
  interests: z
    .array(z.string())
    .min(1, "เลือกความสนใจอย่างน้อย 1 ด้าน")
    .max(MAX_INTERESTS, `เลือกได้ไม่เกิน ${MAX_INTERESTS} ด้าน`),

  // Array Fields — array ของ object (useFieldArray ต้องการ object เพื่อผูก key ให้แต่ละแถว)
  emails: z
    .array(
      z.object({
        // trim ก่อน แล้วค่อยตรวจรูปแบบอีเมล
        address: z.string().trim().pipe(z.email("อีเมลไม่ถูกต้อง")), // ← ตรวจทีละแถว
      }),
    )
    // ─── Array Validation: ตรวจทั้งรายการ ───
    .min(1, "ต้องมีอีเมลอย่างน้อย 1 อีเมล")
    .max(MAX_EMAILS, `มีอีเมลได้ไม่เกิน ${MAX_EMAILS} อีเมล`)
    .refine(
      (items) =>
        new Set(items.map((i) => i.address.toLowerCase())).size ===
        items.length,
      "อีเมลซ้ำกัน",
    ),
});
```

### 14.3 Type จาก schema + กันรหัสซ้ำ

```ts
// ได้ type จาก schema ตรงๆ — ไม่ต้องประกาศ StudentFormValues ซ้ำเอง
export type StudentFormValues = z.infer<typeof studentFormSchema>;

// กันรหัสซ้ำ — ต้องรู้ students ล่าสุดจาก store จึงสร้าง schema ผ่านฟังก์ชัน
export function createStudentFormSchema(existingStudents: Student[]) {
  return studentFormSchema.refine(
    (data) => !existingStudents.some((s) => s.studentId === data.studentId),
    { message: "รหัสนักศึกษานี้มีอยู่แล้ว", path: ["studentId"] },
  );
}
```

---

## ขั้นตอนที่ 15: สร้างฟอร์มเพิ่มนักศึกษาแบบ Dialog ด้วย React Hook Form

สร้างไฟล์ `src/components/students/add-new-student-dialog.tsx` — ใช้ pattern Dialog เดียวกับ
`AddNewCourseDialog` ของ lecture16 (ปุ่ม `DialogTrigger` + `open` state + `<form>` อยู่ใน `DialogContent`)

### 15.1 import และค่าเริ่มต้นของฟอร์ม

วางส่วนนี้ไว้บนสุดของไฟล์

```tsx
import { useMemo, useState } from "react";
import { zodResolver } from "@hookform/resolvers/zod";
import { Plus, RotateCcw, UserPlus, X } from "lucide-react";
import {
  Controller,
  useFieldArray,
  useForm,
  type DefaultValues,
} from "react-hook-form";

import { Button } from "@/components/ui/button";
import { Checkbox } from "@/components/ui/checkbox";
import {
  Dialog,
  DialogContent,
  DialogDescription,
  DialogFooter,
  DialogHeader,
  DialogTitle,
  DialogTrigger,
} from "@/components/ui/dialog";
import {
  Field,
  FieldContent,
  FieldDescription,
  FieldError,
  FieldGroup,
  FieldLabel,
  FieldLegend,
  FieldSet,
} from "@/components/ui/field";
import { Input } from "@/components/ui/input";
import {
  Select,
  SelectContent,
  SelectItem,
  SelectTrigger,
  SelectValue,
} from "@/components/ui/select";
import { useEnrollmentStore } from "@/lib/enrollment-store";
import {
  createStudentFormSchema,
  interestOptions,
  MAX_EMAILS,
  type StudentFormValues,
} from "@/lib/schemas/student-schema";

const programOptions = [
  { value: "CPE", label: "CPE — วิศวกรรมคอมพิวเตอร์" },
  { value: "ISNE", label: "ISNE — วิศวกรรมระบบสารสนเทศและเครือข่าย" },
];

const emptyStudentForm: DefaultValues<StudentFormValues> = {
  studentId: "",
  firstName: "",
  lastName: "",
  program: undefined, // ยังไม่เลือก → Select แสดง placeholder
  interests: [],
  emails: [{ address: "" }], // เริ่มด้วย 1 แถวว่าง
};
```

### 15.2 สร้าง component และตั้งค่าฟอร์มด้วย `useForm`

```tsx
export function AddNewStudentDialog() {
  const addStudent = useEnrollmentStore((s) => s.addStudent);
  const students = useEnrollmentStore((s) => s.students);
  const [open, setOpen] = useState(false);

  // schema ต้องสร้างใหม่เมื่อ students เปลี่ยน เพื่อให้ .refine() กันรหัสซ้ำเห็นข้อมูลล่าสุด
  const schema = useMemo(() => createStudentFormSchema(students), [students]);

  const form = useForm<StudentFormValues>({
    resolver: zodResolver(schema), // ← ใช้ Zod ตรวจ
    defaultValues: emptyStudentForm,
    mode: "onBlur", // ตรวจตอนออกจากช่อง (กวนน้อยกว่า onChange)
  });

  // useFieldArray / onSubmit / resetForm — ดูขั้นตอน 15.5–15.7

  return (
    <Dialog
      open={open}
      onOpenChange={(next) => {
        setOpen(next);
        // ปิด popup แล้วล้างค่า/error — เปิดใหม่ต้องได้ฟอร์มว่าง
        if (!next) resetForm();
      }}
    >
      <DialogTrigger render={<Button />}>
        <UserPlus className="h-4 w-4" />
        เพิ่มนักศึกษา
      </DialogTrigger>
      <DialogContent className="max-h-[90vh] overflow-y-auto sm:max-w-2xl">
        <form
          onSubmit={form.handleSubmit(onSubmit)}
          noValidate
          className="grid gap-4"
        >
          <DialogHeader>
            <DialogTitle>เพิ่มนักศึกษาใหม่</DialogTitle>
            <DialogDescription>
              ลองเว้นช่องว่าง ใส่รหัสนักศึกษาไม่ครบ 9 หลัก ใส่รหัสที่มีอยู่แล้ว
              หรือไม่เลือกความสนใจเลย แล้วกดบันทึก
            </DialogDescription>
          </DialogHeader>

          <FieldGroup className="gap-4">
            {/* ช่องต่างๆ — ดูขั้นตอน 15.3–15.5 */}
          </FieldGroup>

          <DialogFooter>
            {/* ปุ่มล้าง / บันทึก — ดูขั้นตอน 15.7 */}
          </DialogFooter>
        </form>
      </DialogContent>
    </Dialog>
  );
}
```

> 💡 **`DefaultValues<StudentFormValues>`:** `program` ใน schema เป็น `"CPE" | "ISNE"` เท่านั้น แต่ตอนเปิดฟอร์มยังไม่ได้เลือก
> `DefaultValues` ยอมให้ค่าเริ่มต้นเป็น `undefined` ได้ — Zod จะฟ้อง "เลือกหลักสูตร" ถ้ากดบันทึกโดยยังไม่เลือก
>
> 💡 **`mode`:** เลือกได้ว่าจะตรวจเมื่อไร — `"onSubmit"` (กดบันทึก), `"onBlur"` (ออกจากช่อง), `"onChange"` (ทุกครั้งที่พิมพ์),
> `"onTouched"` (ออกจากช่องครั้งแรก แล้วตรวจทุกครั้งที่พิมพ์)

### 15.3 Input และ Select ด้วย `<Controller>`

```tsx
<Controller
  name="studentId"
  control={form.control}
  render={({ field, fieldState }) => (
    <Field data-invalid={fieldState.invalid}>
      <FieldLabel htmlFor="studentId">รหัสนักศึกษา</FieldLabel>
      <Input
        {...field}
        id="studentId"
        placeholder="650610099"
        aria-invalid={fieldState.invalid}
      />
      {fieldState.invalid && <FieldError errors={[fieldState.error]} />}
    </Field>
  )}
/>
```

```tsx
<Controller
  name="program"
  control={form.control}
  render={({ field, fieldState }) => (
    <Field data-invalid={fieldState.invalid}>
      <FieldLabel htmlFor="program">หลักสูตร</FieldLabel>
      <Select
        name={field.name}
        items={programOptions}
        value={field.value ?? null}
        onValueChange={(v) => {
          field.onChange(v);
          field.onBlur(); // Select ไม่มี blur ชัดเจน — ถือว่าแตะแล้วตั้งแต่เลือก
        }}
      >
        <SelectTrigger
          id="program"
          className="w-full"
          aria-invalid={fieldState.invalid}
          ref={field.ref}
        >
          <SelectValue placeholder="เลือกหลักสูตร" />
        </SelectTrigger>
        <SelectContent>
          {programOptions.map((o) => (
            <SelectItem key={o.value} value={o.value}>
              {o.label}
            </SelectItem>
          ))}
        </SelectContent>
      </Select>
      {fieldState.invalid && <FieldError errors={[fieldState.error]} />}
    </Field>
  )}
/>
```

```tsx
<Controller
  name="firstName"
  control={form.control}
  render={({ field, fieldState }) => (
    <Field data-invalid={fieldState.invalid}>
      <FieldLabel htmlFor="firstName">ชื่อ</FieldLabel>
      <Input {...field} id="firstName" aria-invalid={fieldState.invalid} />
      {fieldState.invalid && <FieldError errors={[fieldState.error]} />}
    </Field>
  )}
/>
```

```tsx
<Controller
  name="lastName"
  control={form.control}
  render={({ field, fieldState }) => (
    <Field data-invalid={fieldState.invalid}>
      <FieldLabel htmlFor="lastName">นามสกุล</FieldLabel>
      <Input {...field} id="lastName" aria-invalid={fieldState.invalid} />
      {fieldState.invalid && <FieldError errors={[fieldState.error]} />}
    </Field>
  )}
/>
```

### 15.4 Checkbox ความสนใจ (หลายตัว → array)

```tsx
<Controller
  name="interests"
  control={form.control}
  render={({ field, fieldState }) => (
    <FieldSet data-invalid={fieldState.invalid}>
      <FieldLegend variant="label">ความสนใจ (Checkbox หลายตัว)</FieldLegend>
      <FieldDescription>เลือก 1–3 ด้าน</FieldDescription>
      <FieldGroup data-slot="checkbox-group" className="gap-3">
        {interestOptions.map((item) => (
          <Field
            key={item.id}
            orientation="horizontal"
            data-invalid={fieldState.invalid}
          >
            <Checkbox
              id={`interest-${item.id}`}
              checked={field.value.includes(item.id)}
              onCheckedChange={(checked) => {
                field.onChange(
                  checked
                    ? [...field.value, item.id] // ติ๊ก → เพิ่ม id
                    : field.value.filter((id) => id !== item.id), // เอาออก → กรอง id ทิ้ง
                );
                field.onBlur();
              }}
            />
            <FieldLabel htmlFor={`interest-${item.id}`} className="font-normal">
              {item.label}
            </FieldLabel>
          </Field>
        ))}
      </FieldGroup>
      {fieldState.invalid && <FieldError errors={[fieldState.error]} />}
    </FieldSet>
  )}
/>
```

> 💡 ทั้งกลุ่มใช้ `<Controller>` **ตัวเดียว** (`name="interests"`) — ค่าเป็น array ของ id แล้วแต่ละ Checkbox เพิ่ม/ลบ id ของตัวเองเข้า array

### 15.5 อีเมลแบบเพิ่ม/ลบแถวด้วย `useFieldArray`

```tsx
const { fields, append, remove } = useFieldArray({
  control: form.control,
  name: "emails",
});

// error ระดับ array — อยู่ที่ .root เมื่อมีแถวอยู่แล้ว หรือที่ตัว array เองเมื่อว่าง
const emailsError =
  form.formState.errors.emails?.root ?? form.formState.errors.emails;
```

```tsx
<FieldSet data-invalid={!!emailsError?.message}>
  <FieldLegend variant="label">อีเมล</FieldLegend>
  <FieldDescription>
    {fields.length}/{MAX_EMAILS} อีเมล — ห้ามซ้ำกัน
  </FieldDescription>

  <FieldGroup className="gap-3">
    {/* key ต้องใช้ item.id (ที่ useFieldArray สร้างให้) ไม่ใช่ index */}
    {fields.map((item, index) => (
      <div key={item.id} className="flex items-start gap-2">
        <Controller
          name={`emails.${index}.address`} // ← ผูกกับแถวที่ index
          control={form.control}
          render={({ field, fieldState }) => (
            <Field data-invalid={fieldState.invalid} className="flex-1">
              <FieldContent>
                <Input
                  {...field}
                  type="email"
                  placeholder="name@cmu.ac.th"
                  aria-invalid={fieldState.invalid}
                />
                {fieldState.invalid && (
                  <FieldError errors={[fieldState.error]} />
                )}
              </FieldContent>
            </Field>
          )}
        />
        {/* ─── remove(index) — ต้องเหลืออย่างน้อย 1 แถว ─── */}
        <Button
          type="button"
          variant="ghost"
          size="icon"
          disabled={fields.length <= 1}
          onClick={() => remove(index)}
        >
          <X className="size-4" />
        </Button>
      </div>
    ))}
  </FieldGroup>

  {/* ─── error ระดับ array (.min / .max / .refine) ─── */}
  {emailsError?.message && <FieldError errors={[emailsError]} />}

  {/* ─── append({...}) — ไม่เกิน MAX_EMAILS ─── */}
  <Button
    type="button"
    variant="outline"
    size="sm"
    className="w-fit"
    disabled={fields.length >= MAX_EMAILS}
    onClick={() => append({ address: "" })}
  >
    <Plus className="size-4" />
    เพิ่มอีเมล
  </Button>
</FieldSet>
```

### 15.6 Submit → Zod → Zustand

```tsx
const resetForm = () => form.reset(emptyStudentForm);

// ถูกเรียกเฉพาะเมื่อผ่าน schema แล้วเท่านั้น — ค่าถูก trim แล้ว และ program เป็น "CPE" | "ISNE" แน่นอน
function onSubmit(values: StudentFormValues) {
  addStudent(values);
  resetForm();
  setOpen(false);
}
```

### 15.7 ปุ่มล้างฟอร์ม (Reset)

วางไว้ใน `<DialogFooter>` ข้างปุ่มบันทึก — เรียก `resetForm()` ตัวเดียวกับตอนปิด popup แต่ไม่ปิด popup:

```tsx
<DialogFooter>
  <Button type="button" variant="outline" onClick={resetForm}>
    <RotateCcw className="h-4 w-4" />
    ล้างฟอร์ม
  </Button>
  <Button type="submit">บันทึก</Button>
</DialogFooter>
```

---

## ขั้นตอนที่ 16: ปรับหน้าจัดการนักศึกษา

แก้ `src/pages/admin/students.tsx`

### 16.1 เอาฟอร์มใน Card ออก แล้ววางปุ่ม Dialog ที่หัวหน้า

ลบ state/handler ของฟอร์มเดิม (`values`, `errors`, `touched`, `checkField`, `handleChange`, `handleBlur`, `handleSubmit`,
`errorOf`, `fieldError`, `invalidProps`) และ `<Card>` ทั้งก้อน แล้ววางปุ่มไว้มุมขวาบน:

```tsx
<div className="flex flex-wrap items-start justify-between gap-2">
  <div>
    <h1 className="text-xl font-semibold">จัดการนักศึกษา</h1>
    <p className="text-sm text-muted-foreground">
      {students.length} คน — Lecture 17: รับข้อมูลและตรวจสอบก่อนเข้าสู่ระบบ
    </p>
  </div>
  <div className="flex flex-wrap gap-2">
    <AddNewStudentDialog />
  </div>
</div>
```

### 16.2 เพิ่มคอลัมน์ ความสนใจ / อีเมล / วิชาที่ลงทะเบียน

```tsx
import { interestOptions } from "@/lib/schemas/student-schema";

const interestLabel = (id: string) =>
  interestOptions.find((o) => o.id === id)?.label ?? id;
```

ด้านล่าง `<TableHead>หลักสูตร</TableHead>`

```tsx
<TableHead>ความสนใจ</TableHead>
<TableHead>อีเมล</TableHead>
<TableHead>วิชาที่ลงทะเบียน</TableHead>
```

ด้านล่าง ` <TableCell>{s.program}</TableCell>`

```tsx
<TableCell>
  <div className="flex flex-wrap gap-1">
    {s.interests?.length ? (
      s.interests.map((id) => (
        <Badge key={id} variant="outline">{interestLabel(id)}</Badge>
      ))
    ) : (
      <span className="text-muted-foreground">—</span>
    )}
  </div>
</TableCell>
<TableCell>
  {s.emails?.length ? (
    <div className="flex flex-col gap-0.5 text-sm">
      {s.emails.map((e) => <span key={e.address}>{e.address}</span>)}
    </div>
  ) : (
    <span className="text-muted-foreground">—</span>
  )}
</TableCell>
<TableCell>
  <div className="flex flex-wrap gap-1">
    {enrollments
      .filter((e) => e.studentId === s.studentId)
      .map((e) => (
        <Badge key={e.courseId} variant="secondary">{e.courseId}</Badge>
      ))}
  </div>
</TableCell>
```

## 16.3 สรุปขั้นตอน 16.1-16.2

```tsx
import { FlaskConical } from "lucide-react";

import { ConfirmDeleteButton } from "@/components/confirm-button";
import { AddNewStudentDialog } from "@/components/students/add-new-student-dialog";
import { Badge } from "@/components/ui/badge";
import { Button } from "@/components/ui/button";
import {
  Table,
  TableBody,
  TableCell,
  TableHead,
  TableHeader,
  TableRow,
} from "@/components/ui/table";
import { useEnrollmentStore } from "@/lib/enrollment-store";
import { interestOptions } from "@/lib/schema/student-schema";

const interestLabel = (id: string) =>
  interestOptions.find((o) => o.id === id)?.label ?? id;

/**
 * จัดการนักศึกษา
 * ฟอร์มเพิ่มนักศึกษา + ความสนใจ + อีเมล อยู่ในปุ่ม popup (AddNewStudentDialog)
 * Validate ด้วย React Hook Form + Zod — ดู lib/schemas/student-schema.ts
 */
export default function AdminStudentsPage() {
  const { students, enrollments, addStudent, removeStudent } =
    useEnrollmentStore();

  return (
    <div className="space-y-4">
      <div className="flex flex-wrap items-start justify-between gap-2">
        <div>
          <h1 className="text-xl font-semibold">จัดการนักศึกษา</h1>
          <p className="text-sm text-muted-foreground">
            {students.length} คน — Lecture 17:
            รับข้อมูลและตรวจสอบก่อนเข้าสู่ระบบ
          </p>
        </div>
        <div className="flex flex-wrap gap-2">
          <AddNewStudentDialog />
        </div>
      </div>

      <div className="rounded-lg border">
        <Table>
          <TableHeader>
            <TableRow>
              <TableHead>รหัสนักศึกษา</TableHead>
              <TableHead>ชื่อ</TableHead>
              <TableHead>นามสกุล</TableHead>
              <TableHead>หลักสูตร</TableHead>
              <TableHead>ความสนใจ</TableHead>
              <TableHead>อีเมล</TableHead>
              <TableHead>วิชาที่ลงทะเบียน</TableHead>
              <TableHead className="w-12" />
            </TableRow>
          </TableHeader>
          <TableBody>
            {students.map((s, i) => (
              <TableRow key={`${s.studentId}-${i}`}>
                <TableCell>{s.studentId}</TableCell>
                <TableCell>{s.firstName}</TableCell>
                <TableCell>
                  {s.lastName || <Badge variant="destructive">ว่างเปล่า</Badge>}
                </TableCell>
                <TableCell>{s.program}</TableCell>
                <TableCell>
                  <div className="flex flex-wrap gap-1">
                    {s.interests?.length ? (
                      s.interests.map((id) => (
                        <Badge key={id} variant="outline">
                          {interestLabel(id)}
                        </Badge>
                      ))
                    ) : (
                      <span className="text-muted-foreground">—</span>
                    )}
                  </div>
                </TableCell>
                <TableCell>
                  {s.emails?.length ? (
                    <div className="flex flex-col gap-0.5 text-sm">
                      {s.emails.map((e) => (
                        <span key={e.address}>{e.address}</span>
                      ))}
                    </div>
                  ) : (
                    <span className="text-muted-foreground">—</span>
                  )}
                </TableCell>
                <TableCell>
                  <div className="flex flex-wrap gap-1">
                    {enrollments
                      .filter((e) => e.studentId === s.studentId)
                      .map((e) => (
                        <Badge key={e.courseId} variant="secondary">
                          {e.courseId}
                        </Badge>
                      ))}
                  </div>
                </TableCell>
                <TableCell>
                  <ConfirmDeleteButton
                    label={`ลบ ${s.studentId}`}
                    title="ลบนักศึกษา?"
                    description={`ลบ ${s.studentId} ${s.firstName} ${s.lastName} พร้อมการลงทะเบียนทั้งหมด`}
                    onConfirm={() => removeStudent(s.studentId)}
                  />
                </TableCell>
              </TableRow>
            ))}
          </TableBody>
        </Table>
      </div>
    </div>
  );
}
```

### 16.4 ปุ่มบนหน้าแรก

แก้ `src/pages/home.tsx` ให้มีปุ่มไปหน้าจัดการนักศึกษาด้วย (เดิมมีแค่ปุ่มไปหน้าจัดการการลงทะเบียน):

```tsx
<div className="flex flex-wrap gap-2">
  <Button render={<Link to="/admin/students" />}>ไปหน้าจัดการนักศึกษา</Button>
  <Button variant="outline" render={<Link to="/admin/enrollments" />}>
    ไปหน้าจัดการการลงทะเบียน
  </Button>
</div>
```

---

---

## ขั้นตอนที่ 17:

จากนั้นลบไฟล์ `src/lib/student-validation.ts` ได้เลย — ไม่รียกใช้แล้ว (Zod schema ในขั้นตอนที่ 14 ทำหน้าที่แทน)

## ขั้นตอนที่ 18: สรุปเทียบ — เขียนเอง vs ใช้ React Hook Form + Zod

### 18.1 กฎการตรวจ: `if-else` → Zod schema

**❌ ไม่ใช้ library** — `src/lib/student-validation.ts` ของ lecture16 ให้ตรวจความสนใจ + อีเมล

```ts
// ต้องประกาศ type ของฟอร์มเอง แยกจากกฎการตรวจ
export type StudentFormValues = {
  studentId: string;
  firstName: string;
  lastName: string;
  program: Student["program"] | "";
  interests: string[];
  emails: { address: string }[];
};

// ต้องประกาศ type ของ error เองอีกชุด
export type StudentFormErrors = {
  studentId?: string;
  // ...
  emails?: string;
  emailItems?: (string | undefined)[];
};

export function validateStudentForm(values, existingStudents) {
  const errors: StudentFormErrors = {};
  const studentId = values.studentId.trim();

  if (!/^\d{9}$/.test(studentId)) {
    errors.studentId = "รหัสนักศึกษาต้องเป็นตัวเลข 9 หลัก";
  } else if (existingStudents.some((s) => s.studentId === studentId)) {
    errors.studentId = "รหัสนักศึกษานี้มีอยู่แล้ว";
  }
  if (!values.firstName.trim()) errors.firstName = "กรอกชื่อ";
  // ... ต่ออีกหลายบรรทัด (ความสนใจ, อีเมลทีละแถว, อีเมลทั้งรายการ)
  return errors;
}
```

**✅ ใช้ Zod** — `src/lib/schemas/student-schema.ts` (ขั้นตอนที่ 14)

```ts
import { z } from "zod";
import type { Student } from "@/lib/types";

export const studentFormSchema = z.object({
  studentId: z
    .string()
    .trim()
    .regex(/^\d{9}$/, "รหัสนักศึกษาต้องเป็นตัวเลข 9 หลัก"),
  firstName: z.string().trim().min(1, "กรอกชื่อ"),
  lastName: z.string().trim().min(1, "กรอกนามสกุล"),
  program: z.enum(["CPE", "ISNE"], { message: "เลือกหลักสูตร" }),
  interests: z
    .array(z.string())
    .min(1, "เลือกความสนใจอย่างน้อย 1 ด้าน")
    .max(3, "เลือกได้ไม่เกิน 3 ด้าน"),
  emails: z
    .array(
      z.object({ address: z.string().trim().pipe(z.email("อีเมลไม่ถูกต้อง")) }),
    ) // ← trim แล้วตรวจทีละแถว
    .min(1, "ต้องมีอีเมลอย่างน้อย 1 อีเมล") // ← ตรวจทั้งรายการ
    .max(3, "มีอีเมลได้ไม่เกิน 3 อีเมล")
    .refine(
      (items) =>
        new Set(items.map((i) => i.address.toLowerCase())).size ===
        items.length,
      "อีเมลซ้ำกัน",
    ),
});

// type ได้มาจาก schema — ไม่ต้องประกาศซ้ำ และไม่มีทางไม่ตรงกับกฎ
export type StudentFormValues = z.infer<typeof studentFormSchema>;

// กันรหัสซ้ำ — ต้องรู้ students ล่าสุด จึงสร้าง schema ผ่านฟังก์ชัน
export function createStudentFormSchema(existingStudents: Student[]) {
  return studentFormSchema.refine(
    (data) => !existingStudents.some((s) => s.studentId === data.studentId),
    { message: "รหัสนักศึกษานี้มีอยู่แล้ว", path: ["studentId"] },
  );
}
```

### 18.2 State ของฟอร์ม: `useState` → `useForm`

**❌ ไม่ใช้ library**

```tsx
const [values, setValues] = useState<FormState>(emptyStudentForm);
const [submitted, setSubmitted] = useState(false);
const nextEmailKey = useRef(1);

function update<K extends keyof FormState>(key: K, value: FormState[K]) {
  setValues((prev) => ({ ...prev, [key]: value }));
}

const errors = submitted
  ? validateStudentForm(toFormValues(values), students)
  : {};
```

**✅ ใช้ React Hook Form + Zod**

```tsx
import { zodResolver } from "@hookform/resolvers/zod";
import { Controller, useFieldArray, useForm } from "react-hook-form";

const schema = useMemo(() => createStudentFormSchema(students), [students]);

const form = useForm<StudentFormValues>({
  resolver: zodResolver(schema), // ← ใช้ Zod ตรวจ
  defaultValues: {
    studentId: "",
    firstName: "",
    lastName: "",
    program: undefined,
    interests: [],
    emails: [{ address: "" }],
  },
  mode: "onBlur", // ตรวจตอนออกจากช่อง — เปลี่ยนเป็น "onChange" / "onSubmit" ได้บรรทัดเดียว
});
```

> 💡 ไม่ต้องมี `update()`, `submitted`, `nextEmailKey`, `toFormValues()` — `useForm` ถือค่า, error และสถานะ "แตะแล้ว"
> ของทุกช่องไว้ให้ ส่วนจังหวะที่จะแสดง error เลือกได้จาก `mode`

### 18.3 ผูก Input: `value` + `onChange` → `<Controller>`

**❌ ไม่ใช้ library**

```tsx
<Field data-invalid={!!errors.studentId}>
  <FieldLabel htmlFor="studentId">รหัสนักศึกษา</FieldLabel>
  <Input
    id="studentId"
    value={values.studentId}
    onChange={(e) => update("studentId", e.target.value)}
    aria-invalid={!!errors.studentId}
  />
  {errors.studentId && <FieldError errors={[{ message: errors.studentId }]} />}
</Field>
```

**✅ ใช้ React Hook Form**

```tsx
<Controller
  name="studentId"
  control={form.control}
  render={({ field, fieldState }) => (
    <Field data-invalid={fieldState.invalid}>
      <FieldLabel htmlFor="studentId">รหัสนักศึกษา</FieldLabel>
      <Input {...field} id="studentId" aria-invalid={fieldState.invalid} />
      {fieldState.invalid && <FieldError errors={[fieldState.error]} />}
    </Field>
  )}
/>
```

> 💡 `{...field}` ใส่ `value`, `onChange`, `onBlur`, `name`, `ref` ให้ครบในทีเดียว
> และ `fieldState` บอกทั้ง `invalid` และ `error` ของช่องนั้นโดยไม่ต้องไปหยิบจาก object รวม

### 18.4 Checkbox ความสนใจ

**❌ ไม่ใช้ library**

```tsx
<Checkbox
  checked={values.interests.includes(item.id)}
  onCheckedChange={(checked) =>
    update(
      "interests",
      checked
        ? [...values.interests, item.id]
        : values.interests.filter((id) => id !== item.id),
    )
  }
/>
```

**✅ ใช้ React Hook Form** (อยู่ใน `<Controller name="interests" render={({ field }) => ...}>`)

```tsx
<Checkbox
  checked={field.value.includes(item.id)}
  onCheckedChange={(checked) => {
    field.onChange(
      checked
        ? [...field.value, item.id]
        : field.value.filter((id) => id !== item.id),
    );
    field.onBlur(); // Checkbox ไม่มี blur ชัดเจน — บอกว่าแตะแล้ว เพื่อให้ mode "onBlur" ตรวจ
  }}
/>
```

> 💡 ส่วนนี้โค้ดแทบเท่ากัน — Checkbox หลายตัวยังต้องเพิ่ม/ลบค่าใน array เองทั้งสองแบบ ต่างกันแค่เรียก `field.onChange` แทน `update`

### 18.5 อีเมลเพิ่ม/ลบแถว: จัดการ array เอง → `useFieldArray`

**❌ ไม่ใช้ library**

```tsx
// ต้องสร้าง key เองทุกแถว
type EmailRow = { key: number; address: string };

// แก้ค่าแถวเดียว — ต้อง map ทั้ง array
onChange={(e) =>
  update("emails", values.emails.map((r) =>
    r.key === row.key ? { ...r, address: e.target.value } : r
  ))
}

// เพิ่ม / ลบ
update("emails", [...values.emails, { key: nextEmailKey.current++, address: "" }]);
update("emails", values.emails.filter((r) => r.key !== row.key));
```

**✅ ใช้ `useFieldArray`**

```tsx
const { fields, append, remove } = useFieldArray({
  control: form.control,
  name: "emails",
});

{
  fields.map((item, index) => (
    <div key={item.id}>
      {" "}
      {/* ← id ที่ useFieldArray สร้างให้ */}
      <Controller
        name={`emails.${index}.address`} // ← ผูกกับแถวที่ index
        control={form.control}
        render={({ field, fieldState }) => (
          <Field data-invalid={fieldState.invalid}>
            <Input {...field} type="email" aria-invalid={fieldState.invalid} />
            {fieldState.invalid && <FieldError errors={[fieldState.error]} />}
          </Field>
        )}
      />
      <Button type="button" onClick={() => remove(index)}>
        <X className="size-4" />
      </Button>
    </div>
  ));
}

<Button type="button" onClick={() => append({ address: "" })}>
  เพิ่มอีเมล
</Button>;

{
  /* error ระดับ array (.min / .max / .refine) */
}
{
  form.formState.errors.emails?.root && (
    <FieldError errors={[form.formState.errors.emails.root]} />
  );
}
```

> 💡 **ต่างกันตรงไหน:** `useFieldArray` สร้าง `item.id` ให้แต่ละแถวเอง (ไม่ต้องมี `nextEmailKey`)
> แก้ค่าแถวเดียวผ่าน `{...field}` ได้เลย (ไม่ต้อง `.map()` ทั้ง array) และเพิ่ม/ลบด้วย `append()` / `remove(index)`

### 18.6 Submit

**❌ ไม่ใช้ library**

```tsx
function handleSubmit(e: React.FormEvent<HTMLFormElement>) {
  e.preventDefault();
  setSubmitted(true);

  const formValues = toFormValues(values);
  if (hasErrors(validateStudentForm(formValues, students))) return;

  addStudent({
    studentId: formValues.studentId.trim(),
    firstName: formValues.firstName.trim(),
    lastName: formValues.lastName.trim(),
    program: formValues.program as "CPE" | "ISNE", // ต้อง cast เอง
    interests: formValues.interests,
    emails: formValues.emails.map((e) => ({ address: e.address.trim() })),
  });
  resetForm();
  setOpen(false);
}

<form onSubmit={handleSubmit} noValidate>
```

**✅ ใช้ React Hook Form + Zod**

```tsx
// ถูกเรียกเฉพาะเมื่อผ่าน schema แล้วเท่านั้น — values ถูก trim และมี type ถูกต้องแล้ว
function onSubmit(values: StudentFormValues) {
  addStudent(values);
  form.reset();
  setOpen(false);
}

<form onSubmit={form.handleSubmit(onSubmit)} noValidate>
```

> 💡 `form.handleSubmit` ทำ `preventDefault`, ตรวจด้วย Zod, แสดง error และโฟกัสช่องแรกที่ผิดให้
> ถ้าไม่ผ่านจะไม่เรียก `onSubmit` เลย — `program` มี type เป็น `"CPE" | "ISNE"` แล้ว (ไม่ใช่ `""`) จึงไม่ต้อง cast

### 18.8 สรุปเทียบ

| เรื่อง                     | ❌ useState + if-else                     | ✅ React Hook Form + Zod                           |
| -------------------------- | ----------------------------------------- | -------------------------------------------------- |
| กฎการตรวจ                  | `if-else` ใน `validateStudentForm()`      | `z.object({...})` อ่านทีละ field                   |
| Type ของฟอร์ม              | ประกาศเอง (แยกจากกฎ)                      | `z.infer<typeof schema>`                           |
| Type ของ error             | ประกาศเอง (`StudentFormErrors`)           | `form.formState.errors` (มาจาก schema)             |
| State ของค่า               | `useState` + `update()`                   | `useForm({ defaultValues })`                       |
| แสดง error เมื่อไร         | เขียน `submitted` เอง                     | `mode: "onBlur"` / `"onChange"` / `"onSubmit"` ... |
| ผูก input                  | `value` + `onChange` ทุกช่อง              | `<Controller>` + `{...field}`                      |
| Array (อีเมล)              | สร้าง key เอง, `.map()` / `.filter()` เอง | `useFieldArray` → `append` / `remove`              |
| error ของแถว vs ทั้งรายการ | แยก `emailItems` / `emails` เอง           | `errors.emails[i]` / `errors.emails.root`          |
| Trim + cast ตอน submit     | เขียนเอง                                  | schema ทำให้ (`.trim()`, `z.enum`)                 |
| ต้องติดตั้งอะไรเพิ่ม       | ไม่ต้อง                                   | `zod`, `react-hook-form`, `@hookform/resolvers`    |

> 💡 **เลือกใช้แบบไหน:** ฟอร์ม 1–2 ช่องเขียนเองก็พอ แต่พอมีหลายช่อง + กฎข้ามช่อง + array แบบฟอร์มนี้
> โค้ดเขียนเองจะยาวและพลาดง่าย (ลืม trim, key ซ้ำ, type ไม่ตรงกฎ) — library ช่วยตัดงานซ้ำๆ พวกนี้ออก

---
