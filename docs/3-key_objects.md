# Key Objects

[_Documentation generated by Documatic_](https://www.documatic.com)

<!---Documatic-section-main.test-start--->
## main.test

<!---Documatic-section-test-start--->
<!---Documatic-block-main.test-start--->
<details>
	<summary><code>main.test</code> code snippet</summary>

```python
def test(net, memory_data_loader, test_data_loader):
    net.eval()
    (total_top1, total_top5, total_num, feature_bank) = (0.0, 0.0, 0, [])
    with torch.no_grad():
        for (data, _, target) in tqdm(memory_data_loader, desc='Feature extracting'):
            (feature, out) = net(data.cuda(non_blocking=True))
            feature_bank.append(feature)
        feature_bank = torch.cat(feature_bank, dim=0).t().contiguous()
        feature_labels = torch.tensor(memory_data_loader.dataset.labels, device=feature_bank.device)
        test_bar = tqdm(test_data_loader)
        for (data, _, target) in test_bar:
            (data, target) = (data.cuda(non_blocking=True), target.cuda(non_blocking=True))
            (feature, out) = net(data)
            total_num += data.size(0)
            sim_matrix = torch.mm(feature, feature_bank)
            (sim_weight, sim_indices) = sim_matrix.topk(k=k, dim=-1)
            sim_labels = torch.gather(feature_labels.expand(data.size(0), -1), dim=-1, index=sim_indices)
            sim_weight = (sim_weight / temperature).exp()
            one_hot_label = torch.zeros(data.size(0) * k, c, device=sim_labels.device)
            one_hot_label = one_hot_label.scatter(dim=-1, index=sim_labels.view(-1, 1).long(), value=1.0)
            pred_scores = torch.sum(one_hot_label.view(data.size(0), -1, c) * sim_weight.unsqueeze(dim=-1), dim=1)
            pred_labels = pred_scores.argsort(dim=-1, descending=True)
            total_top1 += torch.sum((pred_labels[:, :1] == target.unsqueeze(dim=-1)).any(dim=-1).float()).item()
            total_top5 += torch.sum((pred_labels[:, :5] == target.unsqueeze(dim=-1)).any(dim=-1).float()).item()
            test_bar.set_description('KNN Test Epoch: [{}/{}] Acc@1:{:.2f}% Acc@5:{:.2f}%'.format(epoch, epochs, total_top1 / total_num * 100, total_top5 / total_num * 100))
    return (total_top1 / total_num * 100, total_top5 / total_num * 100)
```
</details>
<!---Documatic-block-main.test-end--->
<!---Documatic-section-test-end--->

# #
<!---Documatic-section-main.test-end--->

<!---Documatic-section-linear.train_val-start--->
## linear.train_val

<!---Documatic-section-train_val-start--->
<!---Documatic-block-linear.train_val-start--->
<details>
	<summary><code>linear.train_val</code> code snippet</summary>

```python
def train_val(net, data_loader, train_optimizer):
    is_train = train_optimizer is not None
    net.train() if is_train else net.eval()
    (total_loss, total_correct_1, total_correct_5, total_num, data_bar) = (0.0, 0.0, 0.0, 0, tqdm(data_loader))
    with torch.enable_grad() if is_train else torch.no_grad():
        for (data, target) in data_bar:
            (data, target) = (data.cuda(non_blocking=True), target.cuda(non_blocking=True))
            out = net(data)
            loss = loss_criterion(out, target)
            if is_train:
                train_optimizer.zero_grad()
                loss.backward()
                train_optimizer.step()
            total_num += data.size(0)
            total_loss += loss.item() * data.size(0)
            prediction = torch.argsort(out, dim=-1, descending=True)
            total_correct_1 += torch.sum((prediction[:, 0:1] == target.unsqueeze(dim=-1)).any(dim=-1).float()).item()
            total_correct_5 += torch.sum((prediction[:, 0:5] == target.unsqueeze(dim=-1)).any(dim=-1).float()).item()
            data_bar.set_description('{} Epoch: [{}/{}] Loss: {:.4f} ACC@1: {:.2f}% ACC@5: {:.2f}%'.format('Train' if is_train else 'Test', epoch, epochs, total_loss / total_num, total_correct_1 / total_num * 100, total_correct_5 / total_num * 100))
    return (total_loss / total_num, total_correct_1 / total_num * 100, total_correct_5 / total_num * 100)
```
</details>
<!---Documatic-block-linear.train_val-end--->
<!---Documatic-section-train_val-end--->

# #
<!---Documatic-section-linear.train_val-end--->

<!---Documatic-section-main.train-start--->
## main.train

<!---Documatic-section-train-start--->
```mermaid
flowchart LR
main.train-->main.get_negative_mask
```

### Object Calls

* main.get_negative_mask

<!---Documatic-block-main.train-start--->
<details>
	<summary><code>main.train</code> code snippet</summary>

```python
def train(net, data_loader, train_optimizer, temperature, debiased, tau_plus):
    net.train()
    (total_loss, total_num, train_bar) = (0.0, 0, tqdm(data_loader))
    for (pos_1, pos_2, target) in train_bar:
        (pos_1, pos_2) = (pos_1.cuda(non_blocking=True), pos_2.cuda(non_blocking=True))
        (feature_1, out_1) = net(pos_1)
        (feature_2, out_2) = net(pos_2)
        out = torch.cat([out_1, out_2], dim=0)
        neg = torch.exp(torch.mm(out, out.t().contiguous()) / temperature)
        mask = get_negative_mask(batch_size).cuda()
        neg = neg.masked_select(mask).view(2 * batch_size, -1)
        pos = torch.exp(torch.sum(out_1 * out_2, dim=-1) / temperature)
        pos = torch.cat([pos, pos], dim=0)
        if debiased:
            N = batch_size * 2 - 2
            Ng = (-tau_plus * N * pos + neg.sum(dim=-1)) / (1 - tau_plus)
            Ng = torch.clamp(Ng, min=N * np.e ** (-1 / temperature))
        else:
            Ng = neg.sum(dim=-1)
        loss = (-torch.log(pos / (pos + Ng))).mean()
        train_optimizer.zero_grad()
        loss.backward()
        train_optimizer.step()
        total_num += batch_size
        total_loss += loss.item() * batch_size
        train_bar.set_description('Train Epoch: [{}/{}] Loss: {:.4f}'.format(epoch, epochs, total_loss / total_num))
    return total_loss / total_num
```
</details>
<!---Documatic-block-main.train-end--->
<!---Documatic-section-train-end--->

# #
<!---Documatic-section-main.train-end--->

<!---Documatic-section-main.get_negative_mask-start--->
## main.get_negative_mask

<!---Documatic-section-get_negative_mask-start--->
<!---Documatic-block-main.get_negative_mask-start--->
<details>
	<summary><code>main.get_negative_mask</code> code snippet</summary>

```python
def get_negative_mask(batch_size):
    negative_mask = torch.ones((batch_size, 2 * batch_size), dtype=bool)
    for i in range(batch_size):
        negative_mask[i, i] = 0
        negative_mask[i, i + batch_size] = 0
    negative_mask = torch.cat((negative_mask, negative_mask), 0)
    return negative_mask
```
</details>
<!---Documatic-block-main.get_negative_mask-end--->
<!---Documatic-section-get_negative_mask-end--->

# #
<!---Documatic-section-main.get_negative_mask-end--->

[_Documentation generated by Documatic_](https://www.documatic.com)