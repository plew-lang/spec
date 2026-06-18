# サンプルコード

## 基本的な構造体と拡張

```plew
struct Calculator {
    mut val current: F64
}

extension BasicMath {
    impl Calculator {
        // 値を返す非破壊メソッド。新しい Calculator を BasicMath ビューで返すので
        // そのままチェーンできる（ビューは戻り値に伝播しないため、明示して返す）。
        fn add(value: F64) -> Calculator#BasicMath {
            return <Calculator current=self.current + value />#BasicMath
        }

        fn result() -> F64 {
            return self.current
        }
    }
}

extension AdvancedMath {
    impl Calculator {
        inout fn power(exponent: F64) {
            self.current = mathPow(base: self.current, exponent: exponent)
        }

        inout fn sqrt() {
            self.current = mathSqrt(value: self.current)
        }
    }
}

async fn main() {
    // BasicMath ── 値を返す流れるような API（非破壊・チェーン可能）。
    val basicResult = <Calculator current=0.0 />#BasicMath
        .add(value: 10.0)
        .add(value: 5.0)
        .result()  // 15.0

    // AdvancedMath ── inout で self を直接変更する。
    // inout の呼び出しには可変束縛 mut val が要る（val 束縛の中身は変更できない）。
    mut val calc = <Calculator current=basicResult />
    calc#AdvancedMath.power(exponent: 2.0)  // current = 225.0
    calc#AdvancedMath.sqrt()                // current = 15.0

    print("Final result: {calc#BasicMath.result()}")  // 15.0
}
```
