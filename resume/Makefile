all:	resume.html

%.html: %.md
	pandoc -f markdown -t html -o $@ -c resume.css --reference-links $<
	tidy -q -wrap 120 -i -m $@

clean: rm -f *.html *.log
