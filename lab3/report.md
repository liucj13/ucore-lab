练习1、给未被映射的地址映射上物理页（完成do_pgfault()函数）
/* do_pgfault – 这是一个用于处理寻页异常的函数
 * @mm         : 控制所有虚拟内存空间
 * @error_code : 错误代码
 * @addr       : 造成内存访问异常的地址
 *
 * 调用路径: trap--> trap_dispatch-->pgfault_handler-->do_pgfault
 //找地址addr的页表是否存在，假如不存在就新建一个
    if ((ptep = get_pte(mm->pgdir, addr, 1)) == NULL) {
        cprintf("get_pte in do_pgfault failed\n");
        goto failed;
    }
    
    if (*ptep == 0) { 
    				  //假如那一页不存在，则分配一页过去，并且建立物理地址与逻辑地址的映射
        if (pgdir_alloc_page(mm->pgdir, addr, perm) == NULL) {
            cprintf("pgdir_alloc_page in do_pgfault failed\n");
            goto failed;
        }
    }
    else { 
    		//假如pte是一个交换分区的入口，则从硬盘导入，并且调用pagr_insert建立逻辑地址与虚拟地址之间的映射关系
        if(swap_init_ok) {
            struct Page *page=NULL;
            if ((ret = swap_in(mm, addr, &page)) != 0) {
                cprintf("swap_in in do_pgfault failed\n");
                goto failed;
            }    
            page_insert(mm->pgdir, page, addr, perm);
            swap_map_swappable(mm, addr, page, 1);
        }
        else {
            cprintf("no swap_init_ok but ptep is %x, failed\n",*ptep);
            goto failed;
        }
   }

2、完善fifo算法
static int
_fifo_map_swappable(struct mm_struct *mm, uintptr_t addr, struct Page *page, int swap_in)
{
    list_entry_t *head=(list_entry_t*) mm->sm_priv;
    list_entry_t *entry=&(page->pra_page_link);
 
    assert(entry != NULL && head != NULL);
//从队尾插入一页
    list_add(head, entry);
    return 0;
}

static int
_fifo_swap_out_victim(struct mm_struct *mm, struct Page ** ptr_page, int in_tick)
{
     list_entry_t *head=(list_entry_t*) mm->sm_priv;
         assert(head != NULL);
     assert(in_tick==0);
     /* Select the victim */
     /*LAB3 EXERCISE 2: YOUR CODE*/ 
     //选择队尾对应页，并且删除
     list_entry_t *le = head->prev;
     assert(head!=le);
     struct Page *p = le2page(le, pra_page_link);
     list_del(le);
     assert(p !=NULL);
     *ptr_page = p;
     return 0;
}

