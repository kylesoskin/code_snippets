# Vets API 526 info gathering and random code snippets

## Disclaimer
These snippets are for information purposes only. Do not run anything anywhere without evaluating and understanding what it is going to do. 
These were all used a few times and not something I thought would be worth polluting the vets-api repo with. If there is something that is going to be used/reused multiple times it is probably better to commit it to the code in a class or as a sidekiq job/class. 

For example, a lot of the classes under `Sidekiq::Form526BackupSubmissionProcess` are ones that I commited to the code because they offer reusable funcitonality that can be used repeatedly, however these individual snippets, that use those classes, are mostly for one-time/transient needs


```
total = Form526Submission.where('id between ? and ?', 2028256, 2043971);nil
```
Get 526 Form Submissions in range of submission ids 



```
puts x.map {|f| ah = Form526Submission.find(f).auth_headers; [ah['va_eauth_pnid'], ah['va_eauth_pid']].to_csv}.join
```
When `x` is an arrary of `Form526Submission` ids, loop over them all, get the `Form526Submission` object's auth headers. From that, grab the filenumber/ssn and participant ID
Was used to make a mapping of fn/ssn to participant ids csv for a subset of claims



```
date = Date.parse('january 01 2022')

Form526Submission.group('date(created_at)').where('id >= 1027488').where('created_at <= ?', 7.days.ago.beginning_of_day).where(submitted_claim_id:nil).where(backup_submitted_claim_id:nil).count(:id).map {|d,v| [d.to_time.to_i*1000, v]}.sort_by{|k,v| k}


Form526Submission.group("DATE_TRUNC('year', created_at)", "DATE_TRUNC('week', created_at)").where('id >= 1027488').where('created_at <= ?', 7.days.ago.beginning_of_day).where(submitted_claim_id:nil).where(backup_submitted_claim_id:nil).count(:id)

```
Random snippets. For all submissions where the id is greater than the one listed, and created more than 1 week ago, count the number with no `submitted_claim_ids` or `backup_submitted_claim_id`
This was probably to get a count, per week, over time, of complete failures. 


```
total = Form526Submission.where('created_at BETWEEN ? AND ?', 7.days.ago.beginning_of_day, 0.day.ago.beginning_of_day); nil
total_count = total.count
exhausted = total.where(submitted_claim_id: nil).size;nil
totally_failed = total.where(submitted_claim_id: nil).where(backup_submitted_claim_id: nil).size;nil

puts <<EOF

Total Submissions: #{total_count}

Total Number of auto-establish Failures: #{exhausted}

   Successful Backup Submissions: #{exhausted - totally_failed}

   Failed Backup Attempts: #{totally_failed}
EOF
```
This is for the weekly numbers pulled for VA peple







```
x=Form526Submission.find(912)
x.form["form526"]["form526"]["disabilities"][0]["name"] = "ooooooooooooooooooo"
ret = EVSS::DisabilityCompensationForm::Service.new(x.auth_headers).get_form526(x.form["form526"].to_json)
```
This was to test (in staging) loading a submission, changing a disability in the payload, and then seeing if we could generate a filled out PDF from EVSS
We could. 

```
subs = Form526Submission.where('id > 1840615').where(submitted_claim_id: nil).where(backup_submitted_claim_id: nil)
info = subs.map{|s|
  statuses = s.form526_job_statuses
  [ s.id, s.created_at,  statuses.map(&:job_class).include?("BackupSubmission"), statuses.last['error_message']]
}.sort_by(&:first)

pp info
pp info.size

######

Form526Submission.where('id > 1840615').where(submitted_claim_id: nil).where(backup_submitted_claim_id: nil).where('created_at <= ?', 1.day.ago).
```
This is ... grabbing error messages and statuses from some claims

```
Form526Submission.where('id > 1666288').where('id < 1722743').where(submitted_claim_id: nil).where(backup_submitted_claim_id: nil).each {|submission| Sidekiq::Form526BackupSubmissionProcess::NonBreakeredForm526BackgroundLoader.perform_async(sub.id) }
```
For each of a subset of claims that meet a certain criteria, [get a pdf representation of the claim, and upload it to s3 ](https://github.com/department-of-veterans-affairs/vets-api/blob/87e6bf17d4340f7c8f8669259a9dc715fdfeb2aa/lib/sidekiq/form526_backup_submission_process/processor.rb#L416)

```
c = ids.size
i = 0
ids.each {|id| 
	puts "#{id} ... #{i+=1} / #{c}"
	pp Sidekiq::Form526BackupSubmissionProcess::NonBreakeredProcessor.new(id).upload_pdf_submission_to_s3 rescue e.to_s
}
```
Something similar to the above, but instead of queuing, do it in the foreground, and catch and print any errors to my stdout/rails console

```
ids = File.read('/tmp/ids.txt').lines.map(&:chomp)
c = ids.size
i = 0
ids.each do |id| 
    begin
        puts "#{id} ... #{i+=1} / #{c}"
        pp Sidekiq::Form526BackupSubmissionProcess::NonBreakeredProcessor.new(id).upload_pdf_submission_to_s3
    rescue => e
      pp e
    end
end
```
Similar to previous



```
batch = Sidekiq::Batch.new
jids = Form526Submission.last(1000).pluck(:id).map {|id|
    batch.jobs do
        Sidekiq::Form526BackupSubmissionProcess::Form526BackgroundLoader.perform_async(id)
    end
}
bid = batch.bid

=====================================
g = x.map {|batch_ids|
    Proc.new{   
        batch = Sidekiq::Batch.new
        batch_ids.each do |id|
            batch.jobs do
                Sidekiq::Form526BackupSubmissionProcess::Form526BackgroundLoader.perform_async(id)
            end
        end
        batch
    }
}
current_batch = g.pop.call
until ! @lock.locked
    puts [@lock, current_batch]
end    
=====================================


ids = Form526Submission.last(1000).pluck(:id)
batch = Sidekiq::Batch.new
throttle = Sidekiq::Limiter.concurrent('Form526BackupSubmission', 32, wait_timeout: 120, lock_timeout: 60)
ids.each do |id|
    batch.jobs do
      throttle.within_limit do
          Sidekiq::Form526BackupSubmissionProcess::NonBreakeredForm526BackgroundLoader.perform_async(id)
      end
    end
end
```
This is a couple different versions/iterations of queuing a large number of claims in a batch with various throttling methods to not wreck EVSS



```
x = Form526Submission.pluck('user_uuid').where(submitted_claim_id: nil).uniq

x.count {|uuid|  Form526Submission.where(user_uuid: uuid).where.not(submitted_claim_id: nil).count == 0
```
Getting all uniq user ids that have submissions that do not have a `submitted_claim_id`



```
all_ids = Set.new
x = Form526Submission.select(:id, :created_at, :user_uuid, :encrypted_kms_key, :auth_headers_json_ciphertext, :form_json_ciphertext).where(submitted_claim_id: nil).find_in_batches;nil
z=0; x.each {|bat| puts "#{Time.now} | BATCH #{z+=1}"; all_ids.merge(bat.map {|b| [b.id, b.created_at, b.user_uuid, b.form_json, b.auth_headers]})};nil
```
This one, picks out certain fields (for perf reasons) where there are no `submitted_claim_id`, and then, in batches, iterates over all of them, adding data from each to a Set.
With some reporting stuff going to stdout to watch it do it and monitor progress. 


```
file = '/tmp/result.yml'
File.write(file, all_ids.to_yaml)
puts Reports::Uploader.get_s3_link(file)
```
This looks like what was done after the collection of the previous data. (Write the Set to a yaml file, then upload it to S3. `Reports::Uploader.get_s3_link(file)` uploads the file to S3 and outputs a presigned url. 



```
all_ids = Set.new
x = Form526Submission.select(:id, :created_at, :user_uuid, :encrypted_kms_key, :auth_headers_json_ciphertext, :form_json_ciphertext).where('submitted_claim_id IS NULL AND id < 1666372').find_in_batches(batch_size: 10000);nil
z=0; x.each {|bat| puts "#{Time.now} | BATCH #{z+=1}"; all_ids.merge(bat.map {|b| [b.id, b.created_at, b.user_uuid, b.form_json, b.auth_headers]})};nil
```
A variation on the previous snippet. 
