docker-unbound
==============

Run unbound in a docker container, behind an LVS router.

### Build
	docker build -t="janfrode/unbound" .

### Run
	docker run --rm=true  -p 53:53 -p 53:53/udp janfrode/unbound
