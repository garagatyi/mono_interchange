# mono_interchange
Спроба розрахувати скільки Монобанк заробляє на моїх покупках.

## Todo
- IC for partial refunds
- IC for international acquiring when currency code differs

## Вимоги
1. jq

## Сумісність

Протестовано на MacOS.
Деякий синтаксис може бути специфічний для MacOS і вимагатиме адаптації для Linux (наприклад утиліти date).

## Обмеження

- Only for Mono API.
- Only for Mono Mastercard.
- Doesn't handle partial and full refunds properly. E.g. when WOG pay charges higher amount for full trunk and then refund difference with actual gas used. In this case, calculation shows higher Mono earnings than actual
- Doesn't take into account if operation is with international acquiring (e.g. pay for Apple music). In this case, calculation shows lower Mono earnings than actual
- Uses assumed lowest possible interchange rate. Thus calculation shows lower Mono earnings than actual.
See comments in around MCC code handling for details.

## Як використовувати:
1. Отримати персональний токен https://api.monobank.ua/
2. Скачати виписку за останні 30 днів:
```shell
token="<my token is here>"
curl -H "X-Token: $token" https://api.monobank.ua/personal/statement/0/$(date -j -v-30d +%s)/$(date +%s) > transactions.json
```
3. Виконати скрипт:
```shell
printf "%-65s | %-15s | %-9s | %-11s | %-12s\n" "Опис" "Сумма" "Кешбек" "Комісія моно" "Моно заробив"
sum=0
cat transactions.json | jq -c '.[]' | while read -r object; do
	description=$(echo "$object" | jq .description)
	mcc=$(echo "$object" | jq .mcc)
	amount=$(echo "$object" | jq .operationAmount)
	cashback=$(echo "$object" | jq .cashbackAmount)
	
	let normAmount=amount
	interchange=0
	
	if [ "$amount" -lt 0 ]; then
		let normAmount=-amount
		if [ "$normAmount" -lt 100 ]; then
			# if less than 100 there is no IC
			interchange=0
		else
			case "$mcc" in
				# special cases with specific (usually lower) IC
				"4829")
					;;
				"4900")
					let "interchange=-amount*0.007"
					;;
				"9311" | "9211" | "9222" | "9223" | "9399")
					let "interchange=-amount*0.0055+1200"
					;;
				"9402" | "4111" | "7523")
					let "interchange=-amount*0.005"
					;;
				*)
					# Common IC for "Contactless terminal".
					# Taken as possible lowest IC because lower values can be ignored:
					# - Merchant UCAF: Mono supports 3ds and this is for cases when issuer doesn't support
					# - Masterpass: I think it is already decommisioned; anyway rarely used
					let "interchange=-amount*0.018"
			esac
		fi
		let transactionEarnings=interchange-cashback
		let sum+=transactionEarnings
	fi
	
	let "normAmount=normAmount/100.0"
	let "normCashback=cashback/100.0"
	let "normInterchange=interchange/100.0"
	let "normTransactionEarnings=transactionEarnings/100.0"
	
	printf "%-65s | %11.2f грн | %5.2f грн | %8.2f грн | %5.2f грн\n" $description $normAmount $normCashback $normInterchange $normTransactionEarnings 
done
let "sum/=100.0"
printf "Усього Mono заробив (приблизно, дивись обмеження в коді): %0.2f гривень" $sum
```

## Додаткові матеріали

Створити CSV зі своїми транзакціями щоб легше було дивитися в застосунку який відкриває spreadsheets.
```shell
jq -r 'map({description,mcc,hold,amount,operationAmount,currencyCode,commissionRate,cashbackAmount}) | (first | keys_unsorted) as $keys | map([to_entries[] | .value]) as $rows | $keys,$rows[] | @csv' transactions.json > transactions.csv
```
