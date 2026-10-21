# تطبيق Flutter لإدارة ومشاركة جهات الاتصال من ملفات Excel
**(Android • Windows • macOS • Linux | العربية RTL • Material 3 • Riverpod • Hive • Clean Architecture)**

---

## 1. هيكلية المشروع (Clean Architecture مبسطة)

```text
excel_contacts_directory/
├── pubspec.yaml
├── assets/
│   └── contacts_2026.xlsx                     # ملف الإكسل المدمج (اختياري)
└── lib/
    ├── main.dart                              # تهيئة Hive و Riverpod و RTL
    ├── core/
    │   ├── theme/app_theme.dart               # ثيم Material 3 وخط Cairo
    │   └── utils/
    │       ├── phone_normalizer.dart          # تطبيع الأرقام المغربية والدولية (+212)
    │       └── vcard_formatter.dart           # بناء vCard 3.0 والرسائل المنسقة
    ├── domain/
    │   ├── entities/contact_entity.dart       # كيان جهة الاتصال الأساسي
    │   └── repositories/contacts_repository.dart
    ├── data/
    │   ├── models/contact_hive_model.dart     # نموذج Hive مع TypeAdapter جاهز
    │   ├── datasources/
    │   │   ├── excel_parser_service.dart      # قراءة جميع أوراق Excel وتحمل الأعمدة الناقصة
    │   │   ├── export_share_service.dart      # مشاركة vCard، واتساب، بريد، تصدير جماعي
    │   │   └── seed_data.dart                 # بيانات أولية جاهزة للتشغيل الفوري
    │   └── repositories/contacts_repository_impl.dart
    └── presentation/
        ├── providers/contacts_providers.dart  # إدارة الحالة والبحث الفوري والفرز بـ Riverpod
        ├── screens/home_screen.dart           # الشاشة الرئيسية المتجاوبة (هاتف + سطح مكتب)
        └── widgets/
            ├── contact_card.dart              # بطاقة جهة الاتصال
            └── contact_share_sheet.dart       # نافذة المشاركة الشاملة + QR Code
```

---

## الملف: `pubspec.yaml`

```yaml
name: excel_contacts_directory
description: تطبيق Flutter متعدد المنصات (Android, Windows, macOS, Linux) لإدارة ومشاركة جهات الاتصال من ملفات Excel.
publish_to: 'none'
version: 1.0.0+1

environment:
  sdk: '>=3.2.0 <4.0.0'

dependencies:
  flutter:
    sdk: flutter
  flutter_localizations:
    sdk: flutter

  # إدارة الحالة (State Management)
  flutter_riverpod: ^2.5.1

  # التخزين المحلي السريع (Local NoSQL Storage)
  hive: ^2.2.3
  hive_flutter: ^1.1.0

  # اختيار وقراءة وتصدير ملفات Excel و CSV
  file_picker: ^8.0.6
  excel: ^4.0.3
  csv: ^6.0.0

  # المشاركة والروابط الخارجية (vCard, WhatsApp, Email, Call)
  share_plus: ^9.0.0
  url_launcher: ^6.3.0
  path_provider: ^2.1.3

  # توليد رمز الاستجابة السريعة (QR Code)
  qr_flutter: ^4.1.0

  # خطوط عربية أنيقة (Cairo)
  google_fonts: ^6.2.1

dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^4.0.0

flutter:
  uses-material-design: true
  assets:
    - assets/
```

---

## الملف: `lib/core/theme/app_theme.dart`

```dart
import 'package:flutter/material.dart';
import 'package:google_fonts/google_fonts.dart';

/// إعدادات الثيم (Material 3) مع ألوان هادئة واحترافية وخط Cairo العربي
class AppTheme {
  static const Color primaryNavy = Color(0xFF0F4C81);
  static const Color accentTeal = Color(0xFF0D9488);
  static const Color surfaceBg = Color(0xFFF4F7FB);

  static ThemeData get lightTheme {
    final baseTextTheme = GoogleFonts.cairoTextTheme();
    final colorScheme = ColorScheme.fromSeed(
      seedColor: primaryNavy,
      primary: primaryNavy,
      secondary: accentTeal,
      surface: Colors.white,
      brightness: Brightness.light,
    );

    return ThemeData(
      useMaterial3: true,
      colorScheme: colorScheme,
      scaffoldBackgroundColor: surfaceBg,
      textTheme: baseTextTheme,
      appBarTheme: AppBarTheme(
        backgroundColor: primaryNavy,
        foregroundColor: Colors.white,
        elevation: 0,
        centerTitle: false,
        titleTextStyle: GoogleFonts.cairo(
          fontSize: 18,
          fontWeight: FontWeight.w700,
          color: Colors.white,
        ),
      ),
      cardTheme: CardTheme(
        color: Colors.white,
        elevation: 0.5,
        margin: EdgeInsets.zero,
        shape: RoundedRectangleBorder(
          borderRadius: BorderRadius.circular(16),
          side: const BorderSide(color: Color(0xFFE2E8F0)),
        ),
      ),
      inputDecorationTheme: InputDecorationTheme(
        filled: true,
        fillColor: Colors.white,
        contentPadding: const EdgeInsets.symmetric(horizontal: 16, vertical: 12),
        border: OutlineInputBorder(
          borderRadius: BorderRadius.circular(14),
          borderSide: const BorderSide(color: Color(0xFFCBD5E1)),
        ),
        enabledBorder: OutlineInputBorder(
          borderRadius: BorderRadius.circular(14),
          borderSide: const BorderSide(color: Color(0xFFCBD5E1)),
        ),
        focusedBorder: OutlineInputBorder(
          borderRadius: BorderRadius.circular(14),
          borderSide: const BorderSide(color: primaryNavy, width: 1.8),
        ),
      ),
    );
  }
}
```

---

## الملف: `lib/core/utils/phone_normalizer.dart`

```dart
/// نتيجة تطبيع رقم الهاتف (الصيغة الدولية + صيغة العرض المحلية)
class NormalizedPhone {
  final String international;
  final String display;
  final bool isValid;

  const NormalizedPhone({
    required this.international,
    required this.display,
    required this.isValid,
  });
}

/// أداة تطبيع أرقام الهواتف المغربية والدولية وإزالة المسافات والكسور العشرية
class PhoneNormalizer {
  static NormalizedPhone normalize(String? rawValue) {
    if (rawValue == null) {
      return const NormalizedPhone(international: '', display: '', isValid: false);
    }

    String str = rawValue.trim();
    // إزالة الكسور العشرية التي يضيفها Excel أحياناً مثل 660517730.0
    if (RegExp(r'^\d+\.0+$').hasMatch(str)) {
      str = str.replaceAll(RegExp(r'\.0+$'), '');
    }

    if (str.isEmpty || str == 'بدون' || str == '-' || str == 'غير متوفر') {
      return const NormalizedPhone(international: '', display: '', isValid: false);
    }

    // الاحتفاظ بالأرقام وعلامة + فقط
    String cleaned = str.replaceAll(RegExp(r'[^\d+]'), '');
    final digitsOnly = cleaned.replaceAll(RegExp(r'\D'), '');
    if (digitsOnly.length < 7) {
      return NormalizedPhone(international: '', display: str, isValid: false);
    }

    // معالجة البادئة الدولية للمغرب (+212) والأرقام الدولية عامة
    if (cleaned.startsWith('00')) {
      cleaned = '+${cleaned.substring(2)}';
    } else if (cleaned.startsWith('212') && cleaned.length == 12) {
      cleaned = '+$cleaned';
    } else if (RegExp(r'^[567]\d{8}$').hasMatch(cleaned)) {
      // رقم مغربي من 9 أرقام حُذف منه الصفر الأول في Excel
      cleaned = '+212$cleaned';
    } else if (RegExp(r'^0[567]\d{8}$').hasMatch(cleaned)) {
      // رقم مغربي محلي من 10 أرقام يبدأ بـ 05 أو 06 أو 07
      cleaned = '+212${cleaned.substring(1)}';
    } else if (!cleaned.startsWith('+') && cleaned.length >= 10) {
      cleaned = '+$cleaned';
    }

    // صيغة عرض مقروءة: 06 XX XX XX XX
    String display = cleaned;
    if (cleaned.startsWith('+212') && cleaned.length == 13) {
      final local = '0${cleaned.substring(4)}';
      display =
          '${local.substring(0, 2)} ${local.substring(2, 4)} ${local.substring(4, 6)} ${local.substring(6, 8)} ${local.substring(8, 10)}';
    }

    return NormalizedPhone(
      international: cleaned,
      display: display,
      isValid: true,
    );
  }

  /// التحقق مما إذا كانت الخلية تشبه رقم هاتف (للتعرف الذكي على الأعمدة)
  static bool looksLikePhone(String? value) {
    if (value == null || value.trim().isEmpty) return false;
    final s = value.trim().replaceAll(RegExp(r'\.0+$'), '').replaceAll(RegExp(r'[\s\-().]'), '');
    return RegExp(r'^(\+?212|00212|0)?[567]\d{8}$').hasMatch(s) ||
        RegExp(r'^\+?\d{8,15}$').hasMatch(s);
  }
}
```

---

## الملف: `lib/core/utils/vcard_formatter.dart`

```dart
import '../../domain/entities/contact_entity.dart';

/// أداة بناء بطاقات vCard 3.0 القياسية والرسائل النصية المنسقة
class VCardFormatter {
  /// توليد نص vCard 3.0 لجهة اتصال واحدة
  static String toSingleVCard(ContactEntity contact) {
    final cleanName = contact.name.replaceAll(RegExp(r'[\r\n]+'), ' ').trim();
    final cleanOrg = contact.organization.replaceAll(RegExp(r'[\r\n]+'), ' ').trim();
    final tel = contact.phone.isNotEmpty ? contact.phone : contact.displayPhone;

    final buffer = StringBuffer()
      ..writeln('BEGIN:VCARD')
      ..writeln('VERSION:3.0')
      ..writeln('FN;CHARSET=UTF-8:$cleanName')
      ..writeln('N;CHARSET=UTF-8:$cleanName;;;;')
      ..writeln('ORG;CHARSET=UTF-8:$cleanOrg (${contact.groupName})');

    if (tel.isNotEmpty) {
      buffer.writeln('TEL;TYPE=CELL,VOICE:$tel');
    }
    if (contact.note.isNotEmpty) {
      buffer.writeln('NOTE;CHARSET=UTF-8:${contact.note}');
    }
    buffer.write('END:VCARD');
    return buffer.toString();
  }

  /// توليد ملف vCard متعدد (.vcf) لمجموعة كاملة دفعة واحدة
  static String toMultiVCard(List<ContactEntity> contacts) {
    return contacts
        .where((c) => c.phone.isNotEmpty || c.displayPhone.isNotEmpty)
        .map(toSingleVCard)
        .join('\r\n');
  }

  /// سطر منسق سريع للنسخ: "الاسم — الهاتف — المؤسسة"
  static String toOneLineSummary(ContactEntity contact) {
    final phonePart = contact.displayPhone.isNotEmpty
        ? contact.displayPhone
        : (contact.phone.isNotEmpty ? contact.phone : 'بدون رقم');
    return '${contact.name} — $phonePart — ${contact.organization}';
  }

  /// رسالة مفصلة ومنسقة للمشاركة عبر WhatsApp أو البريد الإلكتروني
  static String toDetailedMessage(ContactEntity contact, {bool markdownBold = true}) {
    final b = markdownBold ? '*' : '';
    final phonePart = contact.displayPhone.isNotEmpty ? contact.displayPhone : 'غير متوفر';
    final intlPart = contact.phone.isNotEmpty ? ' (${contact.phone})' : '';

    final lines = <String>[
      '📇 ${b}بطاقة جهة اتصال$b',
      '• ${b}الاسم:$b ${contact.name}',
      '• ${b}المؤسسة:$b ${contact.organization}',
      '• ${b}المجموعة:$b ${contact.groupName}',
      '• ${b}الهاتف:$b $phonePart$intlPart',
    ];
    if (contact.note.isNotEmpty) {
      lines.add('• ${b}ملاحظة:$b ${contact.note}');
    }
    return lines.join('\n');
  }
}
```

---

## الملف: `lib/domain/entities/contact_entity.dart`

```dart
/// كيان جهة الاتصال في طبقة الـ Domain (Clean Architecture)
class ContactEntity {
  final String id;
  final int sequence;
  final String name;
  final String phone;        // الرقم الدولي المطبّع (+2126...)
  final String displayPhone; // الرقم المحلي المقروء (06 XX XX XX XX)
  final String organization;
  final String groupName;    // اسم الورقة (Sheet) = المجموعة
  final String note;

  const ContactEntity({
    required this.id,
    required this.sequence,
    required this.name,
    required this.phone,
    required this.displayPhone,
    required this.organization,
    required this.groupName,
    this.note = '',
  });

  bool get hasValidPhone => phone.isNotEmpty;
}
```

---

## الملف: `lib/domain/repositories/contacts_repository.dart`

```dart
import 'dart:typed_data';
import '../entities/contact_entity.dart';

/// واجهة مستودع جهات الاتصال (Repository Interface)
abstract class ContactsRepository {
  /// استرجاع جميع جهات الاتصال المخزنة محلياً في Hive
  Future<List<ContactEntity>> getSavedContacts();

  /// حفظ قائمة جهات الاتصال في قاعدة بيانات Hive
  Future<void> saveContacts(List<ContactEntity> contacts);

  /// قراءة وتحليل ملف Excel (من bytes) وتحويل كل ورقة إلى مجموعة، ثم حفظها
  Future<List<ContactEntity>> importFromExcelBytes(Uint8List bytes);

  /// إضافة جهة اتصال جديدة يدوياً
  Future<void> addContact(ContactEntity contact);

  /// مسح جميع البيانات أو استعادة البيانات الافتراضية
  Future<List<ContactEntity>> resetToDefaultSeedData();
}
```

---

## الملف: `lib/data/models/contact_hive_model.dart`

```dart
import 'package:hive/hive.dart';
import '../../domain/entities/contact_entity.dart';

/// نموذج التخزين في Hive مع محول TypeAdapter مدمج يدوياً
/// (لا يحتاج إلى تشغيل build_runner ليعمل فوراً عند البناء)
class ContactHiveModel extends ContactEntity {
  const ContactHiveModel({
    required super.id,
    required super.sequence,
    required super.name,
    required super.phone,
    required super.displayPhone,
    required super.organization,
    required super.groupName,
    super.note = '',
  });

  factory ContactHiveModel.fromEntity(ContactEntity entity) {
    return ContactHiveModel(
      id: entity.id,
      sequence: entity.sequence,
      name: entity.name,
      phone: entity.phone,
      displayPhone: entity.displayPhone,
      organization: entity.organization,
      groupName: entity.groupName,
      note: entity.note,
    );
  }
}

/// محول Hive المخصص لحفظ واسترجاع ContactHiveModel بكفاءة
class ContactHiveModelAdapter extends TypeAdapter<ContactHiveModel> {
  @override
  final int typeId = 1;

  @override
  ContactHiveModel read(BinaryReader reader) {
    final numOfFields = reader.readByte();
    final fields = <int, dynamic>{
      for (int i = 0; i < numOfFields; i++) reader.readByte(): reader.read(),
    };
    return ContactHiveModel(
      id: (fields[0] as String?) ?? '',
      sequence: (fields[1] as int?) ?? 0,
      name: (fields[2] as String?) ?? '',
      phone: (fields[3] as String?) ?? '',
      displayPhone: (fields[4] as String?) ?? '',
      organization: (fields[5] as String?) ?? '',
      groupName: (fields[6] as String?) ?? '',
      note: (fields[7] as String?) ?? '',
    );
  }

  @override
  void write(BinaryWriter writer, ContactHiveModel obj) {
    writer
      ..writeByte(8)
      ..writeByte(0)
      ..write(obj.id)
      ..writeByte(1)
      ..write(obj.sequence)
      ..writeByte(2)
      ..write(obj.name)
      ..writeByte(3)
      ..write(obj.phone)
      ..writeByte(4)
      ..write(obj.displayPhone)
      ..writeByte(5)
      ..write(obj.organization)
      ..writeByte(6)
      ..write(obj.groupName)
      ..writeByte(7)
      ..write(obj.note);
  }
}
```

---

## الملف: `lib/data/datasources/excel_parser_service.dart`

```dart
import 'dart:typed_data';
import 'package:excel/excel.dart';
import '../../core/utils/phone_normalizer.dart';
import '../../domain/entities/contact_entity.dart';

/// خدمة قراءة وتحليل ملفات Excel (.xlsx) بأي عدد من الأوراق
/// مع تحمل الصفوف والأعمدة الفارغة أو المكررة
class ExcelParserService {
  static const List<String> _nameAliases = [
    'اسم صاحب الرقم',
    'الاسم والنسب',
    'إسم صاحب الرقم',
    'الإسم والنسب',
    'اسم المستفيد',
    'الاسم الكامل',
    'الاسم',
    'المدير',
    'المسؤول',
    'name',
  ];

  static const List<String> _phoneAliases = [
    'رقم الهاتف',
    'الهاتف',
    'رقم الجوال',
    'الجوال',
    'النقال',
    'الرقم',
    'هاتف',
    'phone',
    'mobile',
  ];

  static const List<String> _orgAliases = [
    'اسم المؤسسة',
    'المؤسسة',
    'إسم المؤسسة',
    'المدرسة',
    'الثانوية',
    'الجهة',
    'الإدارة',
    'organization',
    'school',
  ];

  static const List<String> _noteAliases = [
    'ملاحظة',
    'ملاحظات',
    'بيان',
    'الصفة',
    'note',
  ];

  static const List<String> _seqAliases = [
    'ر.ت.',
    'ر.ت',
    'رت',
    'الرقم الترتيبي',
    'م',
    'ت',
    '#',
    'id',
  ];

  /// قراءة ملف Excel وتحويل كل ورقة (Sheet) إلى مجموعة (groupName)
  List<ContactEntity> parseBytes(Uint8List bytes) {
    final excel = Excel.decodeBytes(bytes);
    if (excel.tables.isEmpty) {
      throw const FormatException('ملف Excel لا يحتوي على أي أوراق عمل صالحة.');
    }

    final allContacts = <ContactEntity>[];

    for (final entry in excel.tables.entries) {
      final sheetName = entry.key.trim();
      final sheet = entry.value;
      final rows = sheet.rows;
      if (rows.isEmpty) continue;

      int headerRowIdx = -1;
      int seqCol = -1;
      int nameCol = -1;
      int phoneCol = -1;
      int orgCol = -1;
      int noteCol = -1;

      // 1. البحث عن صف العناوين في أول 10 صفوف
      for (int r = 0; r < rows.length && r < 10; r++) {
        final rowCells = rows[r].map(_cellToString).toList();
        final nonEmpty = rowCells.where((c) => c.isNotEmpty).length;
        if (nonEmpty < 2) continue;

        int matched = 0;
        for (int c = 0; c < rowCells.length; c++) {
          final text = rowCells[c].toLowerCase();
          if (text.isEmpty) continue;

          if (nameCol == -1 && _nameAliases.any((a) => text.contains(a.toLowerCase()))) {
            nameCol = c;
            matched++;
          } else if (phoneCol == -1 && _phoneAliases.any((a) => text.contains(a.toLowerCase()))) {
            phoneCol = c;
            matched++;
          } else if (orgCol == -1 && _orgAliases.any((a) => text.contains(a.toLowerCase()))) {
            orgCol = c;
            matched++;
          } else if (seqCol == -1 && _seqAliases.any((a) => text == a.toLowerCase())) {
            seqCol = c;
          } else if (noteCol == -1 && _noteAliases.any((a) => text.contains(a.toLowerCase()))) {
            noteCol = c;
          }
        }

        if (matched >= 1) {
          headerRowIdx = r;
          break;
        }
      }

      final startRow = headerRowIdx != -1 ? headerRowIdx + 1 : 0;

      // 2. إذا لم يُكتشف عمود الهاتف بالاسم (مثلاً سُمّي "ملاحظة" في ورقة ابتدائي)، نفحص القيم الرقمية
      if (phoneCol == -1) {
        int bestCol = -1;
        int maxHits = 0;
        final maxCols = rows
            .skip(startRow)
            .take(25)
            .fold<int>(0, (prev, el) => el.length > prev ? el.length : prev);

        for (int c = 0; c < maxCols; c++) {
          if (c == nameCol || c == orgCol || c == seqCol) continue;
          int hits = 0;
          for (int r = startRow; r < rows.length && r < startRow + 25; r++) {
            if (c < rows[r].length && PhoneNormalizer.looksLikePhone(_cellToString(rows[r][c]))) {
              hits++;
            }
          }
          if (hits > maxHits) {
            maxHits = hits;
            bestCol = c;
          }
        }

        if (bestCol != -1 && maxHits >= 1) {
          if (noteCol == bestCol) {
            noteCol = bestCol + 1;
          }
          phoneCol = bestCol;
        }
      }

      // 3. استخراج جهات الاتصال من الصفوف
      int autoSeq = 1;
      for (int r = startRow; r < rows.length; r++) {
        final row = rows[r];
        if (row.isEmpty) continue;

        final nameVal = _safeCell(row, nameCol);
        final orgVal = _safeCell(row, orgCol);
        final rawPhoneVal = _safeCell(row, phoneCol);
        final noteVal = _safeCell(row, noteCol);

        if (nameVal.isEmpty && orgVal.isEmpty && rawPhoneVal.isEmpty) continue;
        if (_nameAliases.contains(nameVal) || _orgAliases.contains(orgVal)) continue;

        final normPhone = PhoneNormalizer.normalize(rawPhoneVal);
        int seqNum = autoSeq;
        final seqRaw = _safeCell(row, seqCol).replaceAll('.0', '');
        final parsedSeq = int.tryParse(seqRaw);
        if (parsedSeq != null) {
          seqNum = parsedSeq;
        }

        String finalNote = noteVal;
        if (!normPhone.isValid && rawPhoneVal == 'بدون' && finalNote.isEmpty) {
          finalNote = 'بدون هاتف';
        }

        allContacts.add(
          ContactEntity(
            id: '${sheetName}_${r}_$autoSeq',
            sequence: seqNum,
            name: nameVal.isNotEmpty ? nameVal : 'غير محدد',
            phone: normPhone.international,
            displayPhone: normPhone.display,
            organization: orgVal.isNotEmpty ? orgVal : 'غير محدد',
            groupName: sheetName,
            note: finalNote,
          ),
        );
        autoSeq++;
      }
    }

    if (allContacts.isEmpty) {
      throw const FormatException('لم يتم العثور على جهات اتصال داخل الملف.');
    }

    return allContacts;
  }

  String _safeCell(List<Data?> row, int colIdx) {
    if (colIdx < 0 || colIdx >= row.length) return '';
    return _cellToString(row[colIdx]);
  }

  String _cellToString(Data? cell) {
    if (cell == null || cell.value == null) return '';
    final val = cell.value.toString().trim();
    if (val.endsWith('.0')) {
      return val.substring(0, val.length - 2);
    }
    return val;
  }
}
```

---

## الملف: `lib/data/datasources/export_share_service.dart`

```dart
import 'dart:convert';
import 'dart:io';
import 'package:csv/csv.dart';
import 'package:excel/excel.dart';
import 'package:flutter/services.dart';
import 'package:path_provider/path_provider.dart';
import 'package:share_plus/share_plus.dart';
import 'package:url_launcher/url_launcher.dart';

import '../../core/utils/vcard_formatter.dart';
import '../../domain/entities/contact_entity.dart';

/// خدمة المشاركة والتصدير (vCard, WhatsApp, Email, Clipboard, Excel, CSV)
class ExportShareService {
  /// 1. مشاركة بطاقة vCard (.vcf) لجهة اتصال واحدة عبر share_plus
  Future<void> shareSingleVCard(ContactEntity contact) async {
    final vcfContent = VCardFormatter.toSingleVCard(contact);
    final safeName = contact.name.replaceAll(RegExp(r'[^\w\u0600-\u06FF]+'), '_');
    final tempDir = await getTemporaryDirectory();
    final file = File('${tempDir.path}/$safeName.vcf');
    await file.writeAsString(vcfContent, encoding: utf8);

    await Share.shareXFiles(
      [XFile(file.path, mimeType: 'text/vcard', name: '$safeName.vcf')],
      subject: 'بطاقة جهة اتصال: ${contact.name}',
      text: VCardFormatter.toOneLineSummary(contact),
    );
  }

  /// 2. إرسال البطاقة المنسقة أو فتح محادثة عبر WhatsApp (wa.me)
  Future<bool> shareViaWhatsApp(ContactEntity contact, {bool directToContact = false}) async {
    final text = Uri.encodeComponent(VCardFormatter.toDetailedMessage(contact));
    String url;

    if (directToContact && contact.phone.isNotEmpty) {
      final digits = contact.phone.replaceAll(RegExp(r'\D'), '');
      final greeting = Uri.encodeComponent('السلام عليكم أستاذ(ة) ${contact.name} (${contact.organization})');
      url = 'https://wa.me/$digits?text=$greeting';
    } else {
      url = 'https://wa.me/?text=$text';
    }

    final uri = Uri.parse(url);
    return launchUrl(uri, mode: LaunchMode.externalApplication);
  }

  /// 3. إرسال عبر البريد الإلكتروني (mailto:)
  Future<bool> shareViaEmail(ContactEntity contact) async {
    final subject = Uri.encodeComponent('بطاقة جهة اتصال: ${contact.name} — ${contact.organization}');
    final body = Uri.encodeComponent(
      VCardFormatter.toDetailedMessage(contact, markdownBold: false),
    );
    final uri = Uri.parse('mailto:?subject=$subject&body=$body');
    return launchUrl(uri);
  }

  /// 4. الاتصال المباشر عبر الهاتف (tel:)
  Future<bool> callNumber(String phone) async {
    if (phone.isEmpty) return false;
    final uri = Uri.parse('tel:$phone');
    return launchUrl(uri);
  }

  /// 5. نسخ نص أو رقم إلى الحافظة (Clipboard)
  Future<void> copyToClipboard(String text) async {
    await Clipboard.setData(ClipboardData(text: text));
  }

  /// 6. تصدير ومشاركة المجموعة كاملة كملف vCard متعدد (.vcf)
  Future<String> exportAndShareGroupVCard(String groupLabel, List<ContactEntity> contacts) async {
    final multiVcf = VCardFormatter.toMultiVCard(contacts);
    final safeGroup = groupLabel.replaceAll(RegExp(r'\s+'), '_');
    final dir = await getTemporaryDirectory();
    final file = File('${dir.path}/مجموعة_$safeGroup.vcf');
    await file.writeAsString(multiVcf, encoding: utf8);

    await Share.shareXFiles(
      [XFile(file.path, mimeType: 'text/vcard')],
      subject: 'جهات اتصال مجموعة $groupLabel (.vcf)',
    );
    return file.path;
  }

  /// 7. تصدير ومشاركة المجموعة كاملة كملف CSV (يدعم العربية مع UTF-8 BOM)
  Future<String> exportAndShareGroupCsv(String groupLabel, List<ContactEntity> contacts) async {
    final rows = <List<dynamic>>[
      ['ر.ت.', 'اسم صاحب الرقم', 'رقم الهاتف', 'الرقم الدولي', 'اسم المؤسسة', 'المجموعة', 'ملاحظة'],
      ...contacts.map((c) => [
            c.sequence,
            c.name,
            c.displayPhone.isNotEmpty ? c.displayPhone : 'بدون',
            c.phone,
            c.organization,
            c.groupName,
            c.note,
          ]),
    ];

    final csvString = const ListToCsvConverter().convert(rows);
    // إضافة UTF-8 BOM ليفتح في Excel بالعربية بشكل سليم
    final bytes = <int>[0xEF, 0xBB, 0xBF, ...utf8.encode(csvString)];

    final safeGroup = groupLabel.replaceAll(RegExp(r'\s+'), '_');
    final dir = await getTemporaryDirectory();
    final file = File('${dir.path}/جهات_اتصال_$safeGroup.csv');
    await file.writeAsBytes(bytes);

    await Share.shareXFiles(
      [XFile(file.path, mimeType: 'text/csv')],
      subject: 'ملف CSV لمجموعة $groupLabel',
    );
    return file.path;
  }

  /// 8. تصدير ومشاركة المجموعة كاملة كملف Excel (.xlsx)
  Future<String> exportAndShareGroupExcel(String groupLabel, List<ContactEntity> contacts) async {
    final excel = Excel.createExcel();
    final defaultSheet = excel.getDefaultSheet();

    // تجميع جهات الاتصال حسب المجموعة لإنشاء ورقة لكل مجموعة
    final byGroup = <String, List<ContactEntity>>{};
    for (final c in contacts) {
      byGroup.putIfAbsent(c.groupName, () => []).add(c);
    }

    for (final entry in byGroup.entries) {
      final sheet = excel[entry.key];
      sheet.appendRow([
        TextCellValue('ر.ت.'),
        TextCellValue('اسم صاحب الرقم'),
        TextCellValue('رقم الهاتف'),
        TextCellValue('الرقم الدولي'),
        TextCellValue('اسم المؤسسة'),
        TextCellValue('ملاحظة'),
      ]);

      for (final c in entry.value) {
        sheet.appendRow([
          IntCellValue(c.sequence),
          TextCellValue(c.name),
          TextCellValue(c.displayPhone.isNotEmpty ? c.displayPhone : 'بدون'),
          TextCellValue(c.phone),
          TextCellValue(c.organization),
          TextCellValue(c.note),
        ]);
      }
    }

    if (defaultSheet != null && !byGroup.containsKey(defaultSheet) && excel.tables.length > 1) {
      excel.delete(defaultSheet);
    }

    final fileBytes = excel.save() ?? <int>[];
    final safeGroup = groupLabel.replaceAll(RegExp(r'\s+'), '_');
    final dir = await getTemporaryDirectory();
    final file = File('${dir.path}/جهات_اتصال_$safeGroup.xlsx');
    await file.writeAsBytes(fileBytes);

    await Share.shareXFiles(
      [
        XFile(
          file.path,
          mimeType: 'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet',
        )
      ],
      subject: 'ملف Excel لمجموعة $groupLabel',
    );
    return file.path;
  }
}
```

---

## الملف: `lib/data/datasources/seed_data.dart`

```dart
import '../../domain/entities/contact_entity.dart';

/// بيانات أولية مدمجة من "تحيين لائحة الهاتف 2026" بثلاث مجموعات (ابتدائي، اعدادي، ثانوي)
/// تعمل تلقائياً عند تشغيل التطبيق لأول مرة أو قبل رفع ملف Excel خارجي.
class SeedData {
  static const List<ContactEntity> initialContacts = [
    // --- مجموعة: ابتدائي ---
    ContactEntity(
      id: 'ابتدائي_1',
      sequence: 1,
      name: 'محمد ابو الرجاء',
      phone: '+212660517730',
      displayPhone: '06 60 51 77 30',
      organization: 'المركب التربوي المندمج تامنصورت ابتدائي',
      groupName: 'ابتدائي',
    ),
    ContactEntity(
      id: 'ابتدائي_2',
      sequence: 2,
      name: 'عبد الله البكاوي',
      phone: '+212662612653',
      displayPhone: '06 62 61 26 53',
      organization: 'مدرسة ابن المقفع',
      groupName: 'ابتدائي',
    ),
    ContactEntity(
      id: 'ابتدائي_3',
      sequence: 3,
      name: 'عز العرب الخبيري',
      phone: '+212660520015',
      displayPhone: '06 60 52 00 15',
      organization: 'مدرسة 20 غشت',
      groupName: 'ابتدائي',
    ),
    ContactEntity(
      id: 'ابتدائي_4',
      sequence: 4,
      name: 'عبد الحميد الغازي',
      phone: '+212660521840',
      displayPhone: '06 60 52 18 40',
      organization: 'مدرسة الحسن الزهراوي',
      groupName: 'ابتدائي',
    ),
    ContactEntity(
      id: 'ابتدائي_5',
      sequence: 5,
      name: 'عبد الكريم الصالحي',
      phone: '+212660526179',
      displayPhone: '06 60 52 61 79',
      organization: 'مدرسة عبد الكريم الخطابي',
      groupName: 'ابتدائي',
    ),
    // --- مجموعة: اعدادي ---
    ContactEntity(
      id: 'اعدادي_1',
      sequence: 1,
      name: 'محمد المنياني',
      phone: '+212661914386',
      displayPhone: '06 61 91 43 86',
      organization: 'الثانوية الاعدادية الماوردي',
      groupName: 'اعدادي',
    ),
    ContactEntity(
      id: 'اعدادي_2',
      sequence: 2,
      name: 'بوميا احمد',
      phone: '+212730800935',
      displayPhone: '07 30 80 09 35',
      organization: 'الثانوية الاعدادية اسعادة',
      groupName: 'اعدادي',
    ),
    ContactEntity(
      id: 'اعدادي_3',
      sequence: 3,
      name: 'خاليد الكاكي',
      phone: '+212661903926',
      displayPhone: '06 61 90 39 26',
      organization: 'الثانوية الاعدادية ابن العريف',
      groupName: 'اعدادي',
    ),
    ContactEntity(
      id: 'اعدادي_4',
      sequence: 4,
      name: 'ياسين أبو الهيثم',
      phone: '+212661056094',
      displayPhone: '06 61 05 60 94',
      organization: 'الثانوية الاعدادية مركز 44',
      groupName: 'اعدادي',
    ),
    ContactEntity(
      id: 'اعدادي_5',
      sequence: 5,
      name: 'محمد البساط',
      phone: '+212661912352',
      displayPhone: '06 61 91 23 52',
      organization: 'الثانوية الاعدادية عبد المومن',
      groupName: 'اعدادي',
    ),
    // --- مجموعة: ثانوي ---
    ContactEntity(
      id: 'ثانوي_1',
      sequence: 1,
      name: 'عبد الغني النعامة',
      phone: '+212655273851',
      displayPhone: '06 55 27 38 51',
      organization: 'الثانوية التاهيلية فاطمة المرنيسي',
      groupName: 'ثانوي',
    ),
    ContactEntity(
      id: 'ثانوي_2',
      sequence: 2,
      name: 'الماحي سعيد',
      phone: '+212654827026',
      displayPhone: '06 54 82 70 26',
      organization: 'الثانوية التأهيلية الكتبية',
      groupName: 'ثانوي',
    ),
    ContactEntity(
      id: 'ثانوي_3',
      sequence: 3,
      name: 'ابتسام لحليمي',
      phone: '+212655273824',
      displayPhone: '06 55 27 38 24',
      organization: 'الثانوية التأهيلية سيدي عبد الرحمان',
      groupName: 'ثانوي',
    ),
    ContactEntity(
      id: 'ثانوي_4',
      sequence: 4,
      name: 'رشيد صبير',
      phone: '+212661914226',
      displayPhone: '06 61 91 42 26',
      organization: 'الثانوية التأهيلية صلاح الدين الايوبي',
      groupName: 'ثانوي',
    ),
    ContactEntity(
      id: 'ثانوي_5',
      sequence: 5,
      name: 'محمد داتش',
      phone: '+212655273852',
      displayPhone: '06 55 27 38 52',
      organization: 'الثانوية التاهيلية رياض الزاهية',
      groupName: 'ثانوي',
    ),
  ];
}
```

---

## الملف: `lib/data/repositories/contacts_repository_impl.dart`

```dart
import 'package:flutter/services.dart';
import 'package:hive/hive.dart';

import '../../domain/entities/contact_entity.dart';
import '../../domain/repositories/contacts_repository.dart';
import '../datasources/excel_parser_service.dart';
import '../datasources/seed_data.dart';
import '../models/contact_hive_model.dart';

/// التنفيذ الفعلي لمستودع جهات الاتصال بالاعتماد على Hive و ExcelParserService
class ContactsRepositoryImpl implements ContactsRepository {
  final Box<ContactHiveModel> box;
  final ExcelParserService excelParser;

  ContactsRepositoryImpl({
    required this.box,
    required this.excelParser,
  });

  @override
  Future<List<ContactEntity>> getSavedContacts() async {
    if (box.isNotEmpty) {
      return box.values.toList();
    }
    return resetToDefaultSeedData();
  }

  @override
  Future<void> saveContacts(List<ContactEntity> contacts) async {
    await box.clear();
    final models = contacts.map(ContactHiveModel.fromEntity).toList();
    await box.addAll(models);
  }

  @override
  Future<List<ContactEntity>> importFromExcelBytes(Uint8List bytes) async {
    final parsed = excelParser.parseBytes(bytes);
    await saveContacts(parsed);
    return parsed;
  }

  @override
  Future<void> addContact(ContactEntity contact) async {
    await box.add(ContactHiveModel.fromEntity(contact));
  }

  @override
  Future<List<ContactEntity>> resetToDefaultSeedData() async {
    try {
      // محاولة قراءة ملف الإكسل الكامل المدمج في assets إن وُجد (385 جهة اتصال)
      final assetBytes = await rootBundle.load('assets/contacts_2026.xlsx');
      final parsed = excelParser.parseBytes(assetBytes.buffer.asUint8List());
      await saveContacts(parsed);
      return parsed;
    } catch (_) {
      // في حال عدم وجود الملف في assets، نستخدم بيانات SeedData المدمجة
      await saveContacts(SeedData.initialContacts);
      return SeedData.initialContacts;
    }
  }
}
```

---

## الملف: `lib/presentation/providers/contacts_providers.dart`

```dart
import 'dart:typed_data';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:hive/hive.dart';

import '../../data/datasources/excel_parser_service.dart';
import '../../data/datasources/export_share_service.dart';
import '../../data/models/contact_hive_model.dart';
import '../../data/repositories/contacts_repository_impl.dart';
import '../../domain/entities/contact_entity.dart';
import '../../domain/repositories/contacts_repository.dart';

/// خيارات الفرز المتاحة
enum ContactSortOption {
  sequenceAsc,
  nameAsc,
  nameDesc,
  organizationAsc,
  organizationDesc,
}

/// مزود صندوق Hive المفتوح في main()
final contactsBoxProvider = Provider<Box<ContactHiveModel>>((ref) {
  throw UnimplementedError('contactsBoxProvider must be overridden in main.dart');
});

/// مزودات الخدمات والمستودع
final excelParserServiceProvider = Provider<ExcelParserService>((ref) {
  return ExcelParserService();
});

final exportShareServiceProvider = Provider<ExportShareService>((ref) {
  return ExportShareService();
});

final contactsRepositoryProvider = Provider<ContactsRepository>((ref) {
  return ContactsRepositoryImpl(
    box: ref.watch(contactsBoxProvider),
    excelParser: ref.watch(excelParserServiceProvider),
  );
});

/// حالة الفلاتر والبحث والفرز
final activeGroupProvider = StateProvider<String>((ref) => 'ALL');
final searchQueryProvider = StateProvider<String>((ref) => '');
final searchAllGroupsProvider = StateProvider<bool>((ref) => false);
final sortOptionProvider = StateProvider<ContactSortOption>(
  (ref) => ContactSortOption.sequenceAsc,
);

/// مزود الحالة الرئيسي لقائمة جهات الاتصال
class ContactsNotifier extends StateNotifier<AsyncValue<List<ContactEntity>>> {
  final ContactsRepository _repository;

  ContactsNotifier(this._repository) : super(const AsyncValue.loading()) {
    loadInitial();
  }

  Future<void> loadInitial() async {
    state = const AsyncValue.loading();
    try {
      final list = await _repository.getSavedContacts();
      state = AsyncValue.data(list);
    } catch (e, st) {
      state = AsyncValue.error(e, st);
    }
  }

  Future<int> importExcelBytes(Uint8List bytes) async {
    state = const AsyncValue.loading();
    try {
      final imported = await _repository.importFromExcelBytes(bytes);
      state = AsyncValue.data(imported);
      return imported.length;
    } catch (e, st) {
      state = AsyncValue.error(e, st);
      rethrow;
    }
  }

  Future<void> addNewContact(ContactEntity contact) async {
    final current = state.valueOrNull ?? <ContactEntity>[];
    await _repository.addContact(contact);
    state = AsyncValue.data([...current, contact]);
  }

  Future<void> resetToDefault() async {
    state = const AsyncValue.loading();
    try {
      final defaults = await _repository.resetToDefaultSeedData();
      state = AsyncValue.data(defaults);
    } catch (e, st) {
      state = AsyncValue.error(e, st);
    }
  }
}

final contactsNotifierProvider =
    StateNotifierProvider<ContactsNotifier, AsyncValue<List<ContactEntity>>>((ref) {
  return ContactsNotifier(ref.watch(contactsRepositoryProvider));
});

/// استخراج قائمة المجموعات (الأوراق) مع عدد جهات الاتصال في كل مجموعة
final groupsMapProvider = Provider<Map<String, int>>((ref) {
  final contacts = ref.watch(contactsNotifierProvider).valueOrNull ?? [];
  final map = <String, int>{};
  for (final c in contacts) {
    map[c.groupName] = (map[c.groupName] ?? 0) + 1;
  }
  return map;
});

/// القائمة المفلترة والمفرزة حسب التبويب النشط والبحث الفوري والفرز
final filteredContactsProvider = Provider<List<ContactEntity>>((ref) {
  final allContacts = ref.watch(contactsNotifierProvider).valueOrNull ?? [];
  final activeGroup = ref.watch(activeGroupProvider);
  final query = ref.watch(searchQueryProvider).trim().toLowerCase();
  final searchAll = ref.watch(searchAllGroupsProvider);
  final sortBy = ref.watch(sortOptionProvider);

  Iterable<ContactEntity> result = allContacts;

  // فلترة حسب المجموعة (إلا إذا كان خيار البحث في كل المجموعات مفعّلاً أثناء الكتابة)
  if (activeGroup != 'ALL' && !(query.isNotEmpty && searchAll)) {
    result = result.where((c) => c.groupName == activeGroup);
  }

  // البحث الفوري بالاسم أو المؤسسة أو الهاتف
  if (query.isNotEmpty) {
    final cleanQueryDigits = query.replaceAll(' ', '');
    result = result.where((c) {
      return c.name.toLowerCase().contains(query) ||
          c.organization.toLowerCase().contains(query) ||
          c.groupName.toLowerCase().contains(query) ||
          c.phone.toLowerCase().contains(cleanQueryDigits) ||
          c.displayPhone.replaceAll(' ', '').contains(cleanQueryDigits);
    });
  }

  final list = result.toList();

  // تطبيق الفرز
  list.sort((a, b) {
    switch (sortBy) {
      case ContactSortOption.nameAsc:
        return a.name.compareTo(b.name);
      case ContactSortOption.nameDesc:
        return b.name.compareTo(a.name);
      case ContactSortOption.organizationAsc:
        return a.organization.compareTo(b.organization);
      case ContactSortOption.organizationDesc:
        return b.organization.compareTo(a.organization);
      case ContactSortOption.sequenceAsc:
        return a.sequence.compareTo(b.sequence);
    }
  });

  return list;
});
```

---

## الملف: `lib/presentation/widgets/contact_share_sheet.dart`

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:qr_flutter/qr_flutter.dart';

import '../../core/theme/app_theme.dart';
import '../../core/utils/vcard_formatter.dart';
import '../../domain/entities/contact_entity.dart';
import '../providers/contacts_providers.dart';

/// نافذة المشاركة الشاملة لجهة اتصال واحدة:
/// - vCard (.vcf) عبر share_plus
/// - WhatsApp عبر wa.me
/// - بريد إلكتروني عبر mailto
/// - نسخ الرقم والنص المنسق
/// - QR Code عبر qr_flutter (لبطاقة vCard أو للرقم فقط)
class ContactShareSheet extends ConsumerStatefulWidget {
  final ContactEntity contact;

  const ContactShareSheet({super.key, required this.contact});

  @override
  ConsumerState<ContactShareSheet> createState() => _ContactShareSheetState();
}

class _ContactShareSheetState extends ConsumerState<ContactShareSheet> {
  bool _qrVCardMode = true;

  void _showSnack(String msg) {
    if (!mounted) return;
    ScaffoldMessenger.of(context).showSnackBar(
      SnackBar(
        content: Text(msg),
        behavior: SnackBarBehavior.floating,
        duration: const Duration(seconds: 2),
      ),
    );
  }

  @override
  Widget build(BuildContext context) {
    final c = widget.contact;
    final shareService = ref.read(exportShareServiceProvider);
    final qrPayload = _qrVCardMode
        ? VCardFormatter.toSingleVCard(c)
        : (c.phone.isNotEmpty ? 'tel:${c.phone}' : c.name);

    return DraggableScrollableSheet(
      initialChildSize: 0.85,
      minChildSize: 0.55,
      maxChildSize: 0.95,
      expand: false,
      builder: (context, scrollController) {
        return Container(
          decoration: const BoxDecoration(
            color: Colors.white,
            borderRadius: BorderRadius.vertical(top: Radius.circular(28)),
          ),
          child: ListView(
            controller: scrollController,
            padding: const EdgeInsets.fromLTRB(20, 12, 20, 28),
            children: [
              // مقبض السحب العلوي
              Center(
                child: Container(
                  width: 44,
                  height: 5,
                  decoration: BoxDecoration(
                    color: Colors.grey.shade300,
                    borderRadius: BorderRadius.circular(10),
                  ),
                ),
              ),
              const SizedBox(height: 16),

              // رأس بيانات جهة الاتصال
              Row(
                children: [
                  CircleAvatar(
                    radius: 26,
                    backgroundColor: AppTheme.primaryNavy,
                    child: Text(
                      c.name.isNotEmpty ? c.name.characters.first : 'ج',
                      style: const TextStyle(
                        color: Colors.white,
                        fontSize: 20,
                        fontWeight: FontWeight.bold,
                      ),
                    ),
                  ),
                  const SizedBox(width: 14),
                  Expanded(
                    child: Column(
                      crossAxisAlignment: CrossAxisAlignment.start,
                      children: [
                        Container(
                          padding: const EdgeInsets.symmetric(horizontal: 8, vertical: 2),
                          decoration: BoxDecoration(
                            color: AppTheme.primaryNavy.withOpacity(0.1),
                            borderRadius: BorderRadius.circular(6),
                          ),
                          child: Text(
                            'مجموعة: ${c.groupName}',
                            style: const TextStyle(
                              fontSize: 11,
                              fontWeight: FontWeight.bold,
                              color: AppTheme.primaryNavy,
                            ),
                          ),
                        ),
                        const SizedBox(height: 4),
                        Text(
                          c.name,
                          style: const TextStyle(fontSize: 17, fontWeight: FontWeight.bold),
                        ),
                        Text(
                          c.organization,
                          style: TextStyle(fontSize: 13, color: Colors.grey.shade700),
                        ),
                        const SizedBox(height: 2),
                        Directionality(
                          textDirection: TextDirection.ltr,
                          child: Text(
                            c.hasValidPhone
                                ? '${c.displayPhone} (${c.phone})'
                                : 'بدون رقم هاتف',
                            style: const TextStyle(
                              fontSize: 13,
                              fontWeight: FontWeight.w700,
                              color: AppTheme.primaryNavy,
                            ),
                          ),
                        ),
                      ],
                    ),
                  ),
                  IconButton(
                    onPressed: () => Navigator.of(context).pop(),
                    icon: const Icon(Icons.close),
                  ),
                ],
              ),
              const Divider(height: 28),

              // 1. زر مشاركة ملف vCard (.vcf) عبر share_plus
              FilledButton.icon(
                style: FilledButton.styleFrom(
                  backgroundColor: AppTheme.primaryNavy,
                  padding: const EdgeInsets.symmetric(vertical: 14),
                  shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(14)),
                ),
                onPressed: () async {
                  await shareService.shareSingleVCard(c);
                },
                icon: const Icon(Icons.contact_page_outlined),
                label: const Text(
                  'مشاركة بطاقة جهة الاتصال vCard (.vcf)',
                  style: TextStyle(fontWeight: FontWeight.bold),
                ),
              ),
              const SizedBox(height: 10),

              // 2. شبكة أزرار المشاركة: WhatsApp، البريد، النسخ، الاتصال
              GridView.count(
                crossAxisCount: 2,
                shrinkWrap: true,
                physics: const NeverScrollableScrollPhysics(),
                mainAxisSpacing: 10,
                crossAxisSpacing: 10,
                childAspectRatio: 2.6,
                children: [
                  _ActionTile(
                    icon: Icons.chat_bubble_outline,
                    color: const Color(0xFF16A34A),
                    title: 'مشاركة عبر واتساب',
                    subtitle: 'wa.me نص منسق',
                    onTap: () => shareService.shareViaWhatsApp(c),
                  ),
                  _ActionTile(
                    icon: Icons.send_to_mobile_outlined,
                    color: AppTheme.accentTeal,
                    title: 'مراسلة الرقم مباشرة',
                    subtitle: 'فتح محادثة واتساب',
                    onTap: () {
                      if (!c.hasValidPhone) {
                        _showSnack('لا يتوفر رقم هاتف صالح لهذه الجهة');
                        return;
                      }
                      shareService.shareViaWhatsApp(c, directToContact: true);
                    },
                  ),
                  _ActionTile(
                    icon: Icons.email_outlined,
                    color: const Color(0xFF0284C7),
                    title: 'البريد الإلكتروني',
                    subtitle: 'إرسال عبر mailto',
                    onTap: () => shareService.shareViaEmail(c),
                  ),
                  _ActionTile(
                    icon: Icons.phone_forwarded_outlined,
                    color: const Color(0xFF059669),
                    title: 'اتصال مباشر',
                    subtitle: c.displayPhone.isNotEmpty ? c.displayPhone : 'غير متوفر',
                    onTap: () => shareService.callNumber(c.phone),
                  ),
                  _ActionTile(
                    icon: Icons.copy_outlined,
                    color: const Color(0xFF475569),
                    title: 'نسخ الرقم فقط',
                    subtitle: 'إلى الحافظة',
                    onTap: () async {
                      if (!c.hasValidPhone) {
                        _showSnack('لا يتوفر رقم هاتف لنسخه');
                        return;
                      }
                      await shareService.copyToClipboard(c.phone);
                      _showSnack('تم نسخ الرقم: ${c.phone}');
                    },
                  ),
                  _ActionTile(
                    icon: Icons.format_quote_outlined,
                    color: const Color(0xFF475569),
                    title: 'نسخ النص المنسق',
                    subtitle: 'الاسم — الهاتف — المؤسسة',
                    onTap: () async {
                      final summary = VCardFormatter.toOneLineSummary(c);
                      await shareService.copyToClipboard(summary);
                      _showSnack('تم نسخ النص المنسق إلى الحافظة');
                    },
                  ),
                ],
              ),
              const SizedBox(height: 18),

              // 3. قسم QR Code عبر مكتبة qr_flutter
              Container(
                padding: const EdgeInsets.all(16),
                decoration: BoxDecoration(
                  color: AppTheme.surfaceBg,
                  borderRadius: BorderRadius.circular(18),
                  border: Border.all(color: const Color(0xFFE2E8F0)),
                ),
                child: Column(
                  children: [
                    Row(
                      mainAxisAlignment: MainAxisAlignment.spaceBetween,
                      children: [
                        const Row(
                          children: [
                            Icon(Icons.qr_code_2, color: AppTheme.primaryNavy),
                            SizedBox(width: 6),
                            Text(
                              'رمز الاستجابة السريعة (QR Code)',
                              style: TextStyle(fontWeight: FontWeight.bold, fontSize: 13),
                            ),
                          ],
                        ),
                        SegmentedButton<bool>(
                          segments: const [
                            ButtonSegment(value: true, label: Text('بطاقة vCard', style: TextStyle(fontSize: 11))),
                            ButtonSegment(value: false, label: Text('الرقم فقط', style: TextStyle(fontSize: 11))),
                          ],
                          selected: {_qrVCardMode},
                          onSelectionChanged: (s) {
                            setState(() => _qrVCardMode = s.first);
                          },
                          style: const ButtonStyle(
                            visualDensity: VisualDensity.compact,
                          ),
                        ),
                      ],
                    ),
                    const SizedBox(height: 14),
                    Container(
                      padding: const EdgeInsets.all(12),
                      decoration: BoxDecoration(
                        color: Colors.white,
                        borderRadius: BorderRadius.circular(16),
                      ),
                      child: QrImageView(
                        data: qrPayload,
                        version: QrVersions.auto,
                        size: 185,
                        backgroundColor: Colors.white,
                        eyeStyle: const QrEyeStyle(
                          eyeShape: QrEyeShape.square,
                          color: AppTheme.primaryNavy,
                        ),
                        dataModuleStyle: const QrDataModuleStyle(
                          dataModuleShape: QrDataModuleShape.square,
                          color: AppTheme.primaryNavy,
                        ),
                      ),
                    ),
                    const SizedBox(height: 8),
                    Text(
                      _qrVCardMode
                          ? 'امسح الرمز بكاميرا الهاتف لحفظ جهة الاتصال كاملةً في هاتفك'
                          : 'امسح الرمز للاتصال المباشر بالرقم',
                      style: TextStyle(fontSize: 12, color: Colors.grey.shade600),
                      textAlign: TextAlign.center,
                    ),
                  ],
                ),
              ),
            ],
          ),
        );
      },
    );
  }
}

class _ActionTile extends StatelessWidget {
  final IconData icon;
  final Color color;
  final String title;
  final String subtitle;
  final VoidCallback onTap;

  const _ActionTile({
    required this.icon,
    required this.color,
    required this.title,
    required this.subtitle,
    required this.onTap,
  });

  @override
  Widget build(BuildContext context) {
    return InkWell(
      onTap: onTap,
      borderRadius: BorderRadius.circular(14),
      child: Container(
        padding: const EdgeInsets.symmetric(horizontal: 12, vertical: 8),
        decoration: BoxDecoration(
          color: color.withOpacity(0.08),
          borderRadius: BorderRadius.circular(14),
          border: Border.all(color: color.withOpacity(0.25)),
        ),
        child: Row(
          children: [
            Icon(icon, color: color, size: 22),
            const SizedBox(width: 10),
            Expanded(
              child: Column(
                mainAxisAlignment: MainAxisAlignment.center,
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  Text(
                    title,
                    maxLines: 1,
                    overflow: TextOverflow.ellipsis,
                    style: TextStyle(
                      fontWeight: FontWeight.bold,
                      fontSize: 12,
                      color: color.withOpacity(0.95),
                    ),
                  ),
                  Text(
                    subtitle,
                    maxLines: 1,
                    overflow: TextOverflow.ellipsis,
                    style: TextStyle(fontSize: 10, color: Colors.grey.shade700),
                  ),
                ],
              ),
            ),
          ],
        ),
      ),
    );
  }
}
```

---

## الملف: `lib/presentation/widgets/contact_card.dart`

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

import '../../core/theme/app_theme.dart';
import '../../domain/entities/contact_entity.dart';
import '../providers/contacts_providers.dart';
import 'contact_share_sheet.dart';

/// بطاقة عرض جهة الاتصال مع أزرار سريعة للاتصال والنسخ والواتساب وفتح قائمة المشاركة
class ContactCard extends ConsumerWidget {
  final ContactEntity contact;

  const ContactCard({super.key, required this.contact});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final shareService = ref.read(exportShareServiceProvider);

    return Card(
      child: InkWell(
        borderRadius: BorderRadius.circular(16),
        onTap: () => _openShareModal(context),
        child: Padding(
          padding: const EdgeInsets.all(14),
          child: Column(
            crossAxisAlignment: CrossAxisAlignment.start,
            children: [
              Row(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  CircleAvatar(
                    radius: 22,
                    backgroundColor: AppTheme.primaryNavy,
                    child: Text(
                      contact.name.isNotEmpty ? contact.name.characters.first : 'ج',
                      style: const TextStyle(color: Colors.white, fontWeight: FontWeight.bold),
                    ),
                  ),
                  const SizedBox(width: 12),
                  Expanded(
                    child: Column(
                      crossAxisAlignment: CrossAxisAlignment.start,
                      children: [
                        Wrap(
                          spacing: 6,
                          runSpacing: 4,
                          crossAxisAlignment: WrapCrossAlignment.center,
                          children: [
                            Container(
                              padding: const EdgeInsets.symmetric(horizontal: 8, vertical: 2),
                              decoration: BoxDecoration(
                                color: AppTheme.primaryNavy.withOpacity(0.1),
                                borderRadius: BorderRadius.circular(6),
                              ),
                              child: Text(
                                contact.groupName,
                                style: const TextStyle(
                                  fontSize: 11,
                                  fontWeight: FontWeight.bold,
                                  color: AppTheme.primaryNavy,
                                ),
                              ),
                            ),
                            Text(
                              '#${contact.sequence}',
                              style: TextStyle(fontSize: 11, color: Colors.grey.shade500),
                            ),
                            if (contact.note.isNotEmpty)
                              Container(
                                padding: const EdgeInsets.symmetric(horizontal: 6, vertical: 2),
                                decoration: BoxDecoration(
                                  color: Colors.amber.shade50,
                                  borderRadius: BorderRadius.circular(6),
                                  border: Border.all(color: Colors.amber.shade200),
                                ),
                                child: Text(
                                  contact.note,
                                  style: TextStyle(fontSize: 10, color: Colors.amber.shade900),
                                ),
                              ),
                          ],
                        ),
                        const SizedBox(height: 4),
                        Text(
                          contact.name,
                          style: const TextStyle(fontSize: 15, fontWeight: FontWeight.bold),
                        ),
                        const SizedBox(height: 2),
                        Row(
                          children: [
                            Icon(Icons.account_balance_outlined, size: 14, color: Colors.grey.shade600),
                            const SizedBox(width: 4),
                            Expanded(
                              child: Text(
                                contact.organization,
                                style: TextStyle(fontSize: 12.5, color: Colors.grey.shade700),
                              ),
                            ),
                          ],
                        ),
                      ],
                    ),
                  ),
                ],
              ),
              const Divider(height: 20),
              Row(
                mainAxisAlignment: MainAxisAlignment.spaceBetween,
                children: [
                  // عرض رقم الهاتف بالاتجاه من اليسار لليمين
                  Row(
                    children: [
                      const Icon(Icons.phone_outlined, size: 16, color: AppTheme.accentTeal),
                      const SizedBox(width: 6),
                      if (contact.hasValidPhone)
                        Directionality(
                          textDirection: TextDirection.ltr,
                          child: Text(
                            contact.displayPhone,
                            style: const TextStyle(
                              fontWeight: FontWeight.w800,
                              fontSize: 13.5,
                              color: AppTheme.primaryNavy,
                            ),
                          ),
                        )
                      else
                        Container(
                          padding: const EdgeInsets.symmetric(horizontal: 8, vertical: 2),
                          decoration: BoxDecoration(
                            color: Colors.orange.shade50,
                            borderRadius: BorderRadius.circular(6),
                          ),
                          child: Text(
                            'بدون رقم هاتف',
                            style: TextStyle(fontSize: 11, color: Colors.orange.shade900),
                          ),
                        ),
                    ],
                  ),

                  // أزرار الإجراءات السريعة والمشاركة
                  Row(
                    mainAxisSize: MainAxisSize.min,
                    children: [
                      if (contact.hasValidPhone) ...[
                        IconButton.filledTonal(
                          visualDensity: VisualDensity.compact,
                          tooltip: 'اتصال مباشر',
                          onPressed: () => shareService.callNumber(contact.phone),
                          icon: const Icon(Icons.call, size: 17),
                        ),
                        IconButton.filledTonal(
                          visualDensity: VisualDensity.compact,
                          tooltip: 'نسخ الرقم',
                          onPressed: () async {
                            await shareService.copyToClipboard(contact.phone);
                            if (context.mounted) {
                              ScaffoldMessenger.of(context).showSnackBar(
                                SnackBar(
                                  content: Text('تم نسخ رقم ${contact.name}'),
                                  behavior: SnackBarBehavior.floating,
                                ),
                              );
                            }
                          },
                          icon: const Icon(Icons.copy, size: 17),
                        ),
                      ],
                      const SizedBox(width: 4),
                      FilledButton.icon(
                        style: FilledButton.styleFrom(
                          backgroundColor: AppTheme.primaryNavy,
                          visualDensity: VisualDensity.compact,
                          padding: const EdgeInsets.symmetric(horizontal: 12, vertical: 8),
                        ),
                        onPressed: () => _openShareModal(context),
                        icon: const Icon(Icons.share, size: 15),
                        label: const Text('مشاركة', style: TextStyle(fontSize: 12)),
                      ),
                    ],
                  ),
                ],
              ),
            ],
          ),
        ),
      ),
    );
  }

  void _openShareModal(BuildContext context) {
    showModalBottomSheet(
      context: context,
      isScrollControlled: true,
      backgroundColor: Colors.transparent,
      builder: (_) => ContactShareSheet(contact: contact),
    );
  }
}
```

---

## الملف: `lib/presentation/screens/home_screen.dart`

```dart
import 'dart:io';
import 'package:file_picker/file_picker.dart';
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

import '../../core/theme/app_theme.dart';
import '../../domain/entities/contact_entity.dart';
import '../providers/contacts_providers.dart';
import '../widgets/contact_card.dart';

/// الشاشة الرئيسية للتطبيق (متجاوبة مع الهواتف وسطح المكتب Windows/macOS/Linux)
class HomeScreen extends ConsumerStatefulWidget {
  const HomeScreen({super.key});

  @override
  ConsumerState<HomeScreen> createState() => _HomeScreenState();
}

class _HomeScreenState extends ConsumerState<HomeScreen> {
  final TextEditingController _searchController = TextEditingController();

  @override
  void dispose() {
    _searchController.dispose();
    super.dispose();
  }

  Future<void> _pickAndImportExcel() async {
    try {
      final result = await FilePicker.platform.pickFiles(
        type: FileType.custom,
        allowedExtensions: ['xlsx'],
        withData: true,
      );
      if (result == null || result.files.isEmpty) return;

      final picked = result.files.single;
      final bytes = picked.bytes ??
          (picked.path != null ? await File(picked.path!).readAsBytes() : null);

      if (bytes == null) {
        _showMessage('تعذر قراءة بيانات الملف المختار', isError: true);
        return;
      }

      final count = await ref.read(contactsNotifierProvider.notifier).importExcelBytes(bytes);
      ref.read(activeGroupProvider.notifier).state = 'ALL';
      _showMessage('تم استيراد $count جهة اتصال وحفظها محلياً في Hive بنجاح!');
    } catch (e) {
      _showMessage('خطأ في قراءة ملف Excel: $e', isError: true);
    }
  }

  void _showMessage(String text, {bool isError = false}) {
    if (!mounted) return;
    ScaffoldMessenger.of(context).showSnackBar(
      SnackBar(
        backgroundColor: isError ? Colors.red.shade800 : AppTheme.primaryNavy,
        behavior: SnackBarBehavior.floating,
        content: Text(text),
      ),
    );
  }

  @override
  Widget build(BuildContext context) {
    final asyncContacts = ref.watch(contactsNotifierProvider);
    final groupsMap = ref.watch(groupsMapProvider);
    final activeGroup = ref.watch(activeGroupProvider);
    final filteredContacts = ref.watch(filteredContactsProvider);
    final searchAll = ref.watch(searchAllGroupsProvider);
    final sortOption = ref.watch(sortOptionProvider);
    final shareService = ref.read(exportShareServiceProvider);

    final totalCount = asyncContacts.valueOrNull?.length ?? 0;
    final activeGroupTitle = activeGroup == 'ALL' ? 'جميع المجموعات' : activeGroup;

    return Scaffold(
      appBar: AppBar(
        title: const Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Text('دليل جهات الاتصال والمؤسسات'),
            Text(
              'إدارة ومشاركة ملفات Excel (Hive + Riverpod)',
              style: TextStyle(fontSize: 11.5, color: Colors.white70),
            ),
          ],
        ),
        actions: [
          // زر رفع ملف Excel
          Padding(
            padding: const EdgeInsets.symmetric(horizontal: 6),
            child: FilledButton.tonalIcon(
              style: FilledButton.styleFrom(
                backgroundColor: Colors.white,
                foregroundColor: AppTheme.primaryNavy,
              ),
              onPressed: _pickAndImportExcel,
              icon: const Icon(Icons.upload_file, size: 18),
              label: const Text('رفع Excel', style: TextStyle(fontWeight: FontWeight.bold)),
            ),
          ),

          // قائمة تصدير المجموعة الكاملة (Excel / vCard متعدد / CSV)
          PopupMenuButton<String>(
            tooltip: 'تصدير ومشاركة المجموعة الحالية',
            icon: const Icon(Icons.ios_share),
            onSelected: (value) async {
              if (filteredContacts.isEmpty) {
                _showMessage('لا توجد جهات اتصال في القائمة الحالية لتصديرها', isError: true);
                return;
              }
              if (value == 'excel') {
                await shareService.exportAndShareGroupExcel(activeGroupTitle, filteredContacts);
              } else if (value == 'vcf') {
                await shareService.exportAndShareGroupVCard(activeGroupTitle, filteredContacts);
              } else if (value == 'csv') {
                await shareService.exportAndShareGroupCsv(activeGroupTitle, filteredContacts);
              } else if (value == 'reset') {
                await ref.read(contactsNotifierProvider.notifier).resetToDefault();
                _showMessage('تمت استعادة البيانات الافتراضية');
              }
            },
            itemBuilder: (context) => [
              PopupMenuItem(
                value: 'excel',
                child: ListTile(
                  leading: const Icon(Icons.table_view, color: Colors.green),
                  title: Text('تصدير المجموعة كـ Excel (.xlsx) (${filteredContacts.length})'),
                  dense: true,
                  contentPadding: EdgeInsets.zero,
                ),
              ),
              PopupMenuItem(
                value: 'vcf',
                child: ListTile(
                  leading: const Icon(Icons.contact_mail, color: AppTheme.primaryNavy),
                  title: Text('تصدير vCard متعدد (.vcf) (${filteredContacts.length})'),
                  dense: true,
                  contentPadding: EdgeInsets.zero,
                ),
              ),
              PopupMenuItem(
                value: 'csv',
                child: ListTile(
                  leading: const Icon(Icons.description_outlined, color: Colors.teal),
                  title: Text('تصدير كملف CSV (${filteredContacts.length})'),
                  dense: true,
                  contentPadding: EdgeInsets.zero,
                ),
              ),
              const PopupMenuDivider(),
              const PopupMenuItem(
                value: 'reset',
                child: ListTile(
                  leading: Icon(Icons.restore, color: Colors.orange),
                  title: Text('استعادة بيانات لائحة 2026 الأصلية'),
                  dense: true,
                  contentPadding: EdgeInsets.zero,
                ),
              ),
            ],
          ),
        ],
      ),
      body: Column(
        children: [
          // 1. شريط المجموعات (الأوراق) + البحث الفوري والفرز
          Container(
            color: Colors.white,
            padding: const EdgeInsets.fromLTRB(16, 12, 16, 12),
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.start,
              children: [
                // تبويبات المجموعات (Sheets)
                SingleChildScrollView(
                  scrollDirection: Axis.horizontal,
                  child: Row(
                    children: [
                      ChoiceChip(
                        label: Text('الكل ($totalCount)'),
                        selected: activeGroup == 'ALL',
                        onSelected: (_) {
                          ref.read(activeGroupProvider.notifier).state = 'ALL';
                        },
                      ),
                      const SizedBox(width: 8),
                      ...groupsMap.entries.map((entry) {
                        final isSelected = activeGroup == entry.key;
                        return Padding(
                          padding: const EdgeInsets.only(left: 8),
                          child: ChoiceChip(
                            label: Text('${entry.key} (${entry.value})'),
                            selected: isSelected,
                            onSelected: (_) {
                              ref.read(activeGroupProvider.notifier).state = entry.key;
                            },
                          ),
                        );
                      }),
                    ],
                  ),
                ),
                const SizedBox(height: 10),

                // حقل البحث الفوري + قائمة الفرز
                Row(
                  children: [
                    Expanded(
                      child: TextField(
                        controller: _searchController,
                        onChanged: (val) {
                          ref.read(searchQueryProvider.notifier).state = val;
                        },
                        decoration: InputDecoration(
                          hintText: 'ابحث بالاسم، المؤسسة، أو رقم الهاتف...',
                          prefixIcon: const Icon(Icons.search),
                          suffixIcon: _searchController.text.isNotEmpty
                              ? IconButton(
                                  icon: const Icon(Icons.clear),
                                  onPressed: () {
                                    _searchController.clear();
                                    ref.read(searchQueryProvider.notifier).state = '';
                                  },
                                )
                              : null,
                        ),
                      ),
                    ),
                    const SizedBox(width: 10),

                    // زر اختيار طريقة الفرز
                    PopupMenuButton<ContactSortOption>(
                      tooltip: 'فرز حسب الاسم أو المؤسسة',
                      initialValue: sortOption,
                      onSelected: (opt) {
                        ref.read(sortOptionProvider.notifier).state = opt;
                      },
                      child: Container(
                        padding: const EdgeInsets.symmetric(horizontal: 14, vertical: 12),
                        decoration: BoxDecoration(
                          color: AppTheme.surfaceBg,
                          borderRadius: BorderRadius.circular(14),
                          border: Border.all(color: const Color(0xFFCBD5E1)),
                        ),
                        child: const Row(
                          children: [
                            Icon(Icons.sort, size: 20, color: AppTheme.primaryNavy),
                            SizedBox(width: 6),
                            Text('فرز', style: TextStyle(fontWeight: FontWeight.bold)),
                          ],
                        ),
                      ),
                      itemBuilder: (_) => const [
                        PopupMenuItem(
                          value: ContactSortOption.sequenceAsc,
                          child: Text('الترتيب الأصلي (ر.ت)'),
                        ),
                        PopupMenuItem(
                          value: ContactSortOption.nameAsc,
                          child: Text('الاسم (أ ← ي)'),
                        ),
                        PopupMenuItem(
                          value: ContactSortOption.nameDesc,
                          child: Text('الاسم (ي ← أ)'),
                        ),
                        PopupMenuItem(
                          value: ContactSortOption.organizationAsc,
                          child: Text('المؤسسة (أ ← ي)'),
                        ),
                        PopupMenuItem(
                          value: ContactSortOption.organizationDesc,
                          child: Text('المؤسسة (ي ← أ)'),
                        ),
                      ],
                    ),
                  ],
                ),

                // شريط خيار البحث في كل المجموعات وأزرار التصدير السريعة للمجموعة
                const SizedBox(height: 8),
                Wrap(
                  spacing: 10,
                  runSpacing: 6,
                  crossAxisAlignment: WrapCrossAlignment.center,
                  alignment: WrapAlignment.spaceBetween,
                  children: [
                    Row(
                      mainAxisSize: MainAxisSize.min,
                      children: [
                        Checkbox(
                          value: searchAll,
                          visualDensity: VisualDensity.compact,
                          onChanged: (v) {
                            ref.read(searchAllGroupsProvider.notifier).state = v ?? false;
                          },
                        ),
                        const Text('البحث في كل المجموعات معاً', style: TextStyle(fontSize: 12)),
                      ],
                    ),
                    Wrap(
                      spacing: 6,
                      children: [
                        ActionChip(
                          avatar: const Icon(Icons.table_chart_outlined, size: 16),
                          label: const Text('تصدير Excel', style: TextStyle(fontSize: 11.5)),
                          onPressed: () => shareService.exportAndShareGroupExcel(
                            activeGroupTitle,
                            filteredContacts,
                          ),
                        ),
                        ActionChip(
                          avatar: const Icon(Icons.contact_phone_outlined, size: 16),
                          label: const Text('تصدير vCard جماعي', style: TextStyle(fontSize: 11.5)),
                          onPressed: () => shareService.exportAndShareGroupVCard(
                            activeGroupTitle,
                            filteredContacts,
                          ),
                        ),
                        ActionChip(
                          avatar: const Icon(Icons.list_alt_outlined, size: 16),
                          label: const Text('تصدير CSV', style: TextStyle(fontSize: 11.5)),
                          onPressed: () => shareService.exportAndShareGroupCsv(
                            activeGroupTitle,
                            filteredContacts,
                          ),
                        ),
                      ],
                    ),
                  ],
                ),
              ],
            ),
          ),

          // 2. قائمة البطاقات المتجاوبة (Grid على الشاشات العريضة لـ Windows/macOS/Linux، و List على الهاتف)
          Expanded(
            child: asyncContacts.when(
              loading: () => const Center(child: CircularProgressIndicator()),
              error: (err, _) => Center(
                child: Padding(
                  padding: const EdgeInsets.all(24),
                  child: Text('حدث خطأ: $err', textAlign: TextAlign.center),
                ),
              ),
              data: (_) {
                if (filteredContacts.isEmpty) {
                  return Center(
                    child: Column(
                      mainAxisSize: MainAxisSize.min,
                      children: [
                        Icon(Icons.search_off_rounded, size: 56, color: Colors.grey.shade400),
                        const SizedBox(height: 10),
                        const Text(
                          'لا توجد جهات اتصال مطابقة للبحث',
                          style: TextStyle(fontWeight: FontWeight.bold, fontSize: 16),
                        ),
                      ],
                    ),
                  );
                }

                return LayoutBuilder(
                  builder: (context, constraints) {
                    final width = constraints.maxWidth;
                    final crossAxisCount = width >= 1100
                        ? 3
                        : width >= 700
                            ? 2
                            : 1;

                    if (crossAxisCount == 1) {
                      return ListView.separated(
                        padding: const EdgeInsets.all(14),
                        itemCount: filteredContacts.length,
                        separatorBuilder: (_, __) => const SizedBox(height: 10),
                        itemBuilder: (context, index) {
                          return ContactCard(contact: filteredContacts[index]);
                        },
                      );
                    }

                    return GridView.builder(
                      padding: const EdgeInsets.all(16),
                      gridDelegate: SliverGridDelegateWithFixedCrossAxisCount(
                        crossAxisCount: crossAxisCount,
                        crossAxisSpacing: 12,
                        mainAxisSpacing: 12,
                        mainAxisExtent: 172,
                      ),
                      itemCount: filteredContacts.length,
                      itemBuilder: (context, index) {
                        return ContactCard(contact: filteredContacts[index]);
                      },
                    );
                  },
                );
              },
            ),
          ),
        ],
      ),
    );
  }
}
```

---

## الملف: `lib/main.dart`

```dart
import 'package:flutter/material.dart';
import 'package:flutter_localizations/flutter_localizations.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:hive_flutter/hive_flutter.dart';

import 'core/theme/app_theme.dart';
import 'data/models/contact_hive_model.dart';
import 'presentation/providers/contacts_providers.dart';
import 'presentation/screens/home_screen.dart';

Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();

  // تهيئة قاعدة البيانات المحلية Hive وتسجيل المحول المخصص
  await Hive.initFlutter();
  if (!Hive.isAdapterRegistered(1)) {
    Hive.registerAdapter(ContactHiveModelAdapter());
  }
  final contactsBox = await Hive.openBox<ContactHiveModel>('contacts_box_v1');

  runApp(
    ProviderScope(
      overrides: [
        contactsBoxProvider.overrideWithValue(contactsBox),
      ],
      child: const ExcelContactsApp(),
    ),
  );
}

class ExcelContactsApp extends StatelessWidget {
  const ExcelContactsApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'دليل جهات الاتصال والمؤسسات',
      debugShowCheckedModeBanner: false,
      theme: AppTheme.lightTheme,
      // تفعيل اللغة العربية واتجاه RTL بشكل افتراضي على جميع المنصات
      locale: const Locale('ar'),
      supportedLocales: const [Locale('ar'), Locale('en')],
      localizationsDelegates: const [
        GlobalMaterialLocalizations.delegate,
        GlobalWidgetsLocalizations.delegate,
        GlobalCupertinoLocalizations.delegate,
      ],
      home: const HomeScreen(),
    );
  }
}
```

---

## إعدادات الصلاحيات وبناء النسخ التنفيذية (APK & Windows/macOS/Linux)

### أولاً: إعداد `android/app/src/main/AndroidManifest.xml` (لمنصة Android 11+)
لضمان عمل روابط `wa.me` و `mailto:` و `tel:` على أندرويد 11 فما فوق، أضف وسم `<queries>` التالي داخل ملف `AndroidManifest.xml` (خارج وسم `<application>`):

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
    <uses-permission android:name="android.permission.INTERNET"/>

    <!-- مطلوب لعمل url_launcher مع واتساب والبريد والاتصال الهاتفي على Android 11+ -->
    <queries>
        <intent>
            <action android:name="android.intent.action.VIEW" />
            <data android:scheme="https" />
        </intent>
        <intent>
            <action android:name="android.intent.action.DIAL" />
            <data android:scheme="tel" />
        </intent>
        <intent>
            <action android:name="android.intent.action.SENDTO" />
            <data android:scheme="mailto" />
        </intent>
    </queries>

    <application
        android:label="دليل الهواتف"
        android:name="${applicationName}"
        android:icon="@mipmap/ic_launcher">
        <!-- باقي إعدادات النشاط الافتراضية -->
    </application>
</manifest>
```

### ثانياً: خطوات تشغيل المشروع وبناء ملف Android APK
```bash
# 1. إنشاء مشروع Flutter جديد (إذا لم تستخدم الأرشيف الجاهز)
flutter create --platforms=android,windows,macos,linux excel_contacts_directory
cd excel_contacts_directory

# 2. بعد نسخ ملف pubspec.yaml ومجلد lib/ وملف assets/:
flutter pub get

# 3. بناء نسخة APK موحدة جاهزة للتثبيت على أي هاتف أندرويد:
flutter build apk --release

# أو بناء حزم مقسمة حسب معمارية المعالج لتقليل الحجم:
flutter build apk --split-per-abi --release
```
- ستجد ملف الـ APK الجاهز في المسار:
  `build/app/outputs/flutter-apk/app-release.apk`

### ثالثاً: خطوات بناء نسخة سطح المكتب (Windows / macOS / Linux)
1. **لبناء نسخة Windows (`.exe`)**:
   - تأكد من تثبيت **Visual Studio 2022** مع حزمة تطوير **Desktop development with C++**.
   - نفّذ الأمرين:
     ```bash
     flutter config --enable-windows-desktop
     flutter build windows --release
     ```
   - ستجد البرنامج الكامل مع ملفاته التنفيذية (`excel_contacts_directory.exe` ومكتبات `.dll` المرافقة) داخل المجلد:
     `build/windows/x64/runner/Release/`

2. **لبناء نسخة macOS أو Linux**:
   ```bash
   # على macOS:
   flutter build macos --release

   # على Linux:
   flutter build linux --release
   ```
