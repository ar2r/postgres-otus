up:
	docker compose up -d

down:
	docker compose down

install:
	#wget https://edu.postgrespro.ru/demo_medium.zip
	#unzip demo_medium.zip
	psql -h localhost -p 5432 -U postgres -d postgres -f demo_medium.sql -c 'alter database demo set search_path to bookings'

